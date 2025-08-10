<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
<title>Speed Master - Traffic Racer</title>
<style>
  body, html {
    margin:0; padding:0; background:#111; color:#eee; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    overflow: hidden;
    user-select:none;
  }
  #gameCanvas {
    display:block;
    margin: 10px auto;
    background: linear-gradient(to top, #444, #222);
    border-radius: 12px;
    box-shadow: 0 0 15px #0f0;
    touch-action:none;
  }
  #startScreen, #gameOverScreen {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.9);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 10;
    color: #0f0;
    text-align: center;
  }
  input, button {
    padding: 12px 18px;
    font-size: 1.2em;
    border-radius: 10px;
    border: none;
    margin-top: 15px;
    background: #0a0;
    color: #eee;
    font-weight: bold;
    cursor: pointer;
    user-select:none;
    transition: background 0.3s ease;
  }
  input:focus, button:hover {
    background: #0f0;
    color: #000;
    outline: none;
  }
  #leaderboardList {
    margin-top: 25px;
    width: 280px;
    max-height: 160px;
    overflow-y: auto;
    background: #111;
    border: 2px solid #0f0;
    border-radius: 10px;
    padding: 10px;
    text-align: left;
  }
  #leaderboardList li {
    margin: 4px 0;
    font-weight: bold;
  }
  #scoreDisplay {
    position: fixed;
    top: 5px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 1.8em;
    font-weight: bold;
    color: #0f0;
    user-select:none;
    z-index: 5;
    text-shadow: 0 0 5px #0f0;
  }
  #instructions {
    position: fixed;
    bottom: 10px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 1em;
    color: #0f0a0a;
    user-select:none;
  }
</style>
</head>
<body>

<div id="startScreen">
  <h1>Speed Master</h1>
  <label for="playerName">Enter your name:</label>
  <input id="playerName" type="text" maxlength="12" placeholder="Your name" autocomplete="off" />
  <button id="startBtn">Start Race</button>

  <h2>Leaderboard</h2>
  <ol id="leaderboardList"></ol>
  <p style="margin-top:10px; font-size:0.85em; color:#080;">Tap left/right side to move your car</p>
</div>

<canvas id="gameCanvas" width="320" height="480" tabindex="0"></canvas>
<div id="scoreDisplay" style="display:none;">Score: 0</div>
<div id="instructions">Tap left or right side to move</div>

<div id="gameOverScreen" style="display:none;">
  <h2>Game Over!</h2>
  <p id="finalScore"></p>
  <button id="restartBtn">Play Again</button>
  <h3>Leaderboard</h3>
  <ol id="gameOverLeaderboard"></ol>
</div>

<audio id="bgMusic" loop src="https://cdn.pixabay.com/download/audio/2022/03/29/audio_ebff882d41.mp3?filename=retro-game-loop-10870.mp3" crossorigin="anonymous"></audio>
<audio id="crashSound" src="https://cdn.pixabay.com/download/audio/2022/03/23/audio_75bb4940e7.mp3?filename=explosion-2-6590.mp3" crossorigin="anonymous"></audio>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  const startScreen = document.getElementById('startScreen');
  const gameOverScreen = document.getElementById('gameOverScreen');
  const leaderboardList = document.getElementById('leaderboardList');
  const gameOverLeaderboard = document.getElementById('gameOverLeaderboard');
  const playerNameInput = document.getElementById('playerName');
  const startBtn = document.getElementById('startBtn');
  const restartBtn = document.getElementById('restartBtn');
  const scoreDisplay = document.getElementById('scoreDisplay');
  const finalScoreText = document.getElementById('finalScore');
  const bgMusic = document.getElementById('bgMusic');
  const crashSound = document.getElementById('crashSound');

  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;
  const laneCount = 3;
  const laneWidth = WIDTH / laneCount;

  // Player car size and position
  const carWidth = 40;
  const carHeight = 70;
  let playerLane = 1;
  let targetLane = 1;
  let playerX = laneToX(playerLane);
  const playerY = HEIGHT - carHeight - 15;

  // Game state
  let traffic = [];
  let speed = 2;
  let score = 0;
  let gameOver = false;
  let explosion = null;

  // Colors
  const playerColor = '#2ecc71';
  const trafficColors = ['#e74c3c', '#f39c12', '#3498db', '#9b59b6', '#e67e22'];

  // Explosion particle count
  const explosionParticleCount = 25;

  // Utilities
  function laneToX(lane) {
    return lane * laneWidth + laneWidth / 2 - carWidth / 2;
  }

  // Draw rounded rectangle helper
  CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
    if (w < 2 * r) r = w / 2;
    if (h < 2 * r) r = h / 2;
    this.beginPath();
    this.moveTo(x + r, y);
    this.arcTo(x + w, y, x + w, y + h, r);
    this.arcTo(x + w, y + h, x, y + h, r);
    this.arcTo(x, y + h, x, y, r);
    this.arcTo(x, y, x + w, y, r);
    this.closePath();
    return this;
  }

  // Draw car with simple windows & stripes
  function drawCar(x, y, width, height, color, isPlayer = false) {
    ctx.fillStyle = color;
    ctx.roundRect(x, y, width, height, 12).fill();

    // Windows & details for player car
    if (isPlayer) {
      ctx.fillStyle = 'rgba(255,255,255,0.6)';
      ctx.roundRect(x + width * 0.15, y + height * 0.15, width * 0.7, height * 0.25, 6).fill();
      ctx.fillStyle = 'rgba(255,255,255,0.3)';
      ctx.fillRect(x + width * 0.35, y + height * 0.5, width * 0.3, height * 0.15);
    }
  }

  // Explosion particle class
  class Particle {
    constructor(x, y) {
      this.x = x;
      this.y = y;
      this.radius = Math.random() * 5 + 2;
      this.color = `hsl(${Math.random() * 60 + 30}, 100%, 50%)`;
      this.speedX = (Math.random() - 0.5) * 8;
      this.speedY = (Math.random() - 0.5) * 8;
      this.alpha = 1;
      this.decay = Math.random() * 0.03 + 0.015;
    }
    update() {
      this.x += this.speedX;
      this.y += this.speedY;
      this.alpha -= this.decay;
      this.radius *= 0.96;
    }
    draw() {
      ctx.save();
      ctx.globalAlpha = this.alpha;
      ctx.fillStyle = this.color;
      ctx.beginPath();
      ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }
  }

  // Explosion object
  class Explosion {
    constructor(x, y) {
      this.particles = [];
      for (let i = 0; i < explosionParticleCount; i++) {
        this.particles.push(new Particle(x, y));
      }
      this.done = false;
    }
    update() {
      this.particles.forEach(p => p.update());
      this.particles = this.particles.filter(p => p.alpha > 0.05 && p.radius > 0.5);
      if (this.particles.length === 0) this.done = true;
    }
    draw() {
      this.particles.forEach(p => p.draw());
    }
  }

  // Spawn a new traffic car avoiding too-close spawns
  function spawnTraffic() {
    let lane;
    if (traffic.length === 0) {
      lane = Math.floor(Math.random() * laneCount);
    } else {
      // Choose lane not too close to last cars near top
      const safeLanes = [0, 1, 2].filter(l => {
        return !traffic.some(c => c.lane === l && c.y < 150);
      });
      lane = safeLanes.length ? safeLanes[Math.floor(Math.random() * safeLanes.length)] : Math.floor(Math.random() * laneCount);
    }
    traffic.push({
      lane: lane,
      x: laneToX(lane),
      y: -carHeight - Math.random() * 150,
      width: carWidth,
      height: carHeight,
      color: trafficColors[Math.floor(Math.random() * trafficColors.length)],
      passed: false,
    });
  }

  // Collision detection between two rects
  function rectsCollide(r1, r2) {
    return !(r1.x > r2.x + r2.width ||
             r1.x + r1.width < r2.x ||
             r1.y > r2.y + r2.height ||
             r1.y + r1.height < r2.y);
  }

  // Update game state
  function update() {
    if (gameOver) {
      if (explosion) {
        explosion.update();
        if (explosion.done) {
          showGameOverScreen();
        }
      }
      return;
    }

    // Smooth player lane movement
    let desiredX = laneToX(targetLane);
    if (Math.abs(playerX - desiredX) < 5) {
      playerX = desiredX;
      playerLane = targetLane;
    } else {
      playerX += (desiredX - playerX) * 0.3;
    }

    // Move traffic cars down
    traffic.forEach(car => {
      car.y += speed;
      if (!car.passed && car.y > playerY + carHeight) {
        car.passed = true;
        score++;
        speed += 0.03; // gradually speed up
      }
    });

    // Remove off-screen cars
    traffic = traffic.filter(car => car.y < HEIGHT + carHeight);

    // Spawn new cars if less than 6
    if (traffic.length < 6 && Math.random() < 0.04 + speed * 0.003) {
      spawnTraffic();
    }

    // Check collisions
    const playerRect = {x: playerX, y: playerY, width: carWidth, height: carHeight};
    for (let car of traffic) {
      let carRect = {x: car.x, y: car.y, width: car.width, height: car.height};
      if (rectsCollide(playerRect, carRect)) {
        gameOver = true;
        crashSound.currentTime = 0;
        crashSound.play();
        explosion = new Explosion(playerX + carWidth / 2, playerY + carHeight / 2);
        bgMusic.pause();
        break;
      }
    }

    scoreDisplay.textContent = `Score: ${score}`;
  }

  // Draw game elements
  function draw() {
    // Draw road background & lanes
    ctx.fillStyle = '#222';
    ctx.fillRect(0, 0, WIDTH, HEIGHT);

    // Draw lane dividers
    ctx.strokeStyle = '#555';
    ctx.lineWidth = 4;
    ctx.setLineDash([20, 20]);
    for (let i = 1; i < laneCount; i++) {
      ctx.beginPath();
      ctx.moveTo(i * laneWidth, 0);
      ctx.lineTo(i * laneWidth, HEIGHT);
      ctx.stroke();
    }
    ctx.setLineDash([]);

    // Draw traffic cars
    traffic.forEach(car => {
      drawCar(car.x, car.y, car.width, car.height, car.color);
    });

    // Draw player car or explosion
    if (!gameOver) {
      drawCar(playerX, playerY, carWidth, carHeight, playerColor, true);
    } else if (explosion) {
      explosion.draw();
    }
  }

  // Game loop
  function gameLoop() {
    update();
    draw();
    if (!gameOver || (explosion && !explosion.done)) {
      requestAnimationFrame(gameLoop);
    }
  }

  // Leaderboard storage helpers
  function loadLeaderboard() {
    const data = localStorage.getItem('speedMasterLeaderboard');
    if (!data) return [];
    try {
      return JSON.parse(data);
    } catch {
      return [];
    }
  }

  function saveLeaderboard(list) {
    localStorage.setItem('speedMasterLeaderboard', JSON.stringify(list));
  }

  function addScoreToLeaderboard(name, score) {
    const list = loadLeaderboard();
    list.push({name, score});
    list.sort((a, b) => b.score - a.score);
    if (list.length > 10) list.length = 10;
    saveLeaderboard(list);
    return list;
  }

  function updateLeaderboardDisplay() {
    const list = loadLeaderboard();
    leaderboardList.innerHTML = list.map(item => `<li><strong>${item.name}</strong>: ${item.score}</li>`).join('');
  }

  function showGameOverScreen() {
    finalScoreText.textContent = `${playerNameInput.value.trim()}, your score: ${score}`;
    startScreen.style.display = 'none';
    gameOverScreen.style.display = 'flex';
    scoreDisplay.style.display = 'none';
    updateGameOverLeaderboard();
  }

  function updateGameOverLeaderboard() {
    const list = loadLeaderboard();
    gameOverLeaderboard.innerHTML = list.map(item => `<li><strong>${item.name}</strong>: ${item.score}</li>`).join('');
  }

  // Start game
  function startGame() {
    const name = playerNameInput.value.trim();
    if (!name) {
      alert('Please enter your name.');
      playerNameInput.focus();
      return;
    }
    playerLane = 1;
    targetLane = 1;
    playerX = laneToX(playerLane);
    traffic = [];
    speed = 2;
    score = 0;
    gameOver = false;
    explosion = null;

    startScreen.style.display = 'none';
    gameOverScreen.style.display = 'none';
    scoreDisplay.style.display = 'block';

    bgMusic.currentTime = 0;
    bgMusic.volume = 0.3;
    bgMusic.play().catch(() => {});

    updateLeaderboardDisplay();
    gameLoop();
  }

  // Restart game after game over
  restartBtn.onclick = () => {
    // Add score to leaderboard
    addScoreToLeaderboard(playerNameInput.value.trim(), score);
    startGame();
  };

  startBtn.onclick = startGame;

  // Controls: tap left/right half screen to switch lane
  canvas.addEventListener('touchstart', e => {
    if (gameOver) return;
    const touchX = e.touches[0].clientX - canvas.getBoundingClientRect().left;
    if (touchX < WIDTH / 2) {
      if (targetLane > 0) targetLane--;
    } else {
      if (targetLane < laneCount - 1) targetLane++;
    }
    e.preventDefault();
  });

  // Keyboard support (desktop)
  window.addEventListener('keydown', e => {
    if (gameOver) return;
    if (e.key === 'ArrowLeft' || e.key === 'a') {
      if (targetLane > 0) targetLane--;
    } else if (e.key === 'ArrowRight' || e.key === 'd') {
      if (targetLane < laneCount - 1) targetLane++;
    }
  });

  // Initial leaderboard fill
  updateLeaderboardDisplay();
})();
</script>

</body>
</html>
