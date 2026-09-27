# Shadow-runner
a simple and basic web game
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>UMBRA RUNNER: SHADOW WALK</title>
  <style>
    * {
      box-sizing: border-box;
      user-select: none;
      -webkit-user-select: none;
    }
    body {
      margin: 0;
      padding: 0;
      background: #06050b;
      color: #fff;
      font-family: 'Trebuchet MS', 'Segoe UI', sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: space-between;
      height: 100vh;
      overflow: hidden;
    }
    #ui-panel {
      width: 100%;
      max-width: 500px;
      padding: 10px 16px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      background: #0f0c21;
      border-bottom: 2px solid #ff0055;
      box-shadow: 0 0 15px rgba(255, 0, 85, 0.4);
    }
    .game-title {
      font-size: 13px;
      font-weight: 900;
      letter-spacing: 2px;
      color: #ff0055;
      text-shadow: 0 0 8px rgba(255,0,85,0.6);
    }
    .stat-box {
      font-size: 12px;
      font-weight: bold;
      color: #8b85b1;
    }
    .stat-val {
      color: #00f0ff;
      font-size: 15px;
    }
    .skin-name {
      color: #ff0055;
      font-size: 13px;
      font-weight: 900;
      letter-spacing: 1px;
    }
    .bar-container {
      width: 100px;
      height: 14px;
      background: #1a1738;
      border-radius: 10px;
      overflow: hidden;
      border: 1px solid #ff0055;
      display: inline-block;
      vertical-align: middle;
    }
    #hp-bar {
      width: 100%;
      height: 100%;
      background: linear-gradient(90deg, #ff0055, #ff5500);
      transition: width 0.1s linear;
    }
    #game-container {
      position: relative;
      width: 100%;
      max-width: 500px;
      flex-grow: 1;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    canvas {
      width: 100%;
      height: 100%;
      max-height: 420px;
      background: #05040a;
      box-shadow: 0 0 30px rgba(0,0,0,0.9);
    }
    #touch-controls {
      width: 100%;
      max-width: 500px;
      height: 100px;
      padding: 10px 14px;
      display: flex;
      background: #0f0c21;
      border-top: 2px solid #00f0ff;
    }
    .touch-btn {
      flex: 1;
      background: #1a153b;
      border: 2px solid #3d3570;
      border-radius: 12px;
      color: #00f0ff;
      font-size: 18px;
      font-weight: 900;
      letter-spacing: 1px;
      display: flex;
      align-items: center;
      justify-content: center;
      touch-action: manipulation;
    }
    .touch-btn:active {
      background: #00f0ff;
      color: #06050b;
      transform: scale(0.97);
    }
  </style>
</head>
<body>

  <div id="ui-panel">
    <div>
      <div class="game-title">UMBRA RUNNER</div>
      <div class="stat-box">BEST: <span id="best-txt" class="stat-val">0</span>m | DIST: <span id="dist-txt" class="stat-val">0</span>m</div>
      <div class="stat-box">FORM: <span id="skin-txt" class="skin-name">Shadow Fiend</span></div>
    </div>
    <div>
      <div class="stat-box">HEALTH</div>
      <div class="bar-container"><div id="hp-bar"></div></div>
    </div>
  </div>

  <div id="game-container">
    <canvas id="gameCanvas" width="500" height="340"></canvas>
  </div>

  <div id="touch-controls">
    <div class="touch-btn" id="btn-forward">DASH FORWARD ▶▶</div>
  </div>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    // Game Variables
    let hp = 100;
    let distance = 0;
    let highScore = localStorage.getItem('umbra_high_score') || 0;
    let moveForward = false;
    let animFrame = 0;

    document.getElementById('best-txt').innerText = highScore;

    // Web Audio Synthesizer for Orb Collect Sound FX
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    function playOrbSound() {
      if (audioCtx.state === 'suspended') audioCtx.resume();
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.connect(gain);
      gain.connect(audioCtx.destination);
      osc.frequency.setValueAtTime(600, audioCtx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(1200, audioCtx.currentTime + 0.1);
      gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.1);
      osc.start();
      osc.stop(audioCtx.currentTime + 0.1);
    }

    // Demon Character
    const demon = { x: 100, y: 220, width: 32, height: 48 };

    let buildings = [];
    let shadows = [];
    let orbs = [];

    let currentX = 0;
    for (let i = 0; i < 4; i++) {
      spawnBuilding(currentX);
    }

    function spawnBuilding(startX) {
      let bWidth = 80 + Math.random() * 40;
      let bHeight = 140 + Math.random() * 60;

      let minGap = 130 + Math.min(260, Math.floor(distance / 4));
      let gap = minGap + Math.random() * 80;

      let bX = startX === 0 ? 40 : startX + gap;
      let bY = 270 - bHeight;

      buildings.push({ x: bX, y: bY, width: bWidth, height: bHeight });

      let shadowWidth = bWidth + 50;
      shadows.push({ x: bX, y: 265, width: shadowWidth, height: 35 });

      // Spawn Health Orbs in sunny gaps
      if (startX !== 0 && Math.random() > 0.3) {
        orbs.push({
          x: bX - (gap / 2),
          y: 245,
          radius: 8,
          collected: false
        });
      }

      currentX = bX + bWidth;
    }

    // Touch / Button Handlers
    const btnForward = document.getElementById('btn-forward');

    btnForward.addEventListener('touchstart', (e) => { e.preventDefault(); moveForward = true; });
    btnForward.addEventListener('touchend', (e) => { e.preventDefault(); moveForward = false; });
    btnForward.addEventListener('mousedown', () => moveForward = true);
    btnForward.addEventListener('mouseup', () => moveForward = false);

    function update() {
      if (moveForward) {
        animFrame++;
        let speed = 5.5;
        distance += 1;
        document.getElementById('dist-txt').innerText = distance;

        // Move World Objects
        for (let i = 0; i < buildings.length; i++) {
          buildings[i].x -= speed;
          shadows[i].x -= speed;
        }
        for (let o of orbs) {
          o.x -= speed;
        }

        // Spawn logic
        let lastB = buildings[buildings.length - 1];
        if (lastB.x < canvas.width) {
          spawnBuilding(lastB.x + lastB.width);
        }

        // Cleanup
        if (buildings[0].x + buildings[0].width < -200) {
          buildings.shift();
          shadows.shift();
        }
      }

      // Check shadow safety
      let inShadow = false;
      for (let s of shadows) {
        if (demon.x + 16 >= s.x && demon.x + 16 <= s.x + s.width) {
          inShadow = true;
          break;
        }
      }

      // Orb Collisions (Orbs now restore HP!)
      for (let o of orbs) {
        if (!o.collected && Math.abs((demon.x + 16) - o.x) < 20) {
          o.collected = true;
          playOrbSound();
          if (hp < 100) hp += 25; // Orbs heal 25 HP
          if (hp > 100) hp = 100;
        }
      }

      // Health Damage / Recovery
      if (!inShadow) {
        hp -= 0.65;
        if (hp <= 0) {
          hp = 0;
          if (distance > highScore) {
            highScore = distance;
            localStorage.setItem('umbra_high_score', highScore);
            document.getElementById('best-txt').innerText = highScore;
            alert(`NEW RECORD! You traveled ${distance}m!`);
          } else {
            alert(`GAME OVER! Distance Covered: ${distance}m`);
          }
          // Reset Game
          hp = 100;
          distance = 0;
          buildings = [];
          shadows = [];
          orbs = [];
          currentX = 0;
          for (let i = 0; i < 4; i++) spawnBuilding(currentX);
        }
      } else {
        if (hp < 100) hp += 0.2;
      }

      document.getElementById('hp-bar').style.width = hp + '%';
    }

    function drawDemon(x, y) {
      // Skin System based on Distance Milestones
      let skinName = "Shadow Fiend";
      let auraColor = 'rgba(255, 0, 85, 0.3)';
      let hornColor = '#ff0055';
      let eyeColor = '#00f0ff';
      let bodyColor = '#120d26';

      if (distance >= 500) {
        skinName = "Solar Eclipse Demon";
        auraColor = 'rgba(255, 215, 0, 0.6)';
        hornColor = '#ffd700';
        eyeColor = '#ffffff';
        bodyColor = '#2b2100';
      } else if (distance >= 350) {
        skinName = "Neon Phantom";
        auraColor = 'rgba(0, 255, 128, 0.5)';
        hornColor = '#00ff80';
        eyeColor = '#aaff00';
        bodyColor = '#002b12';
      } else if (distance >= 200) {
        skinName = "Void Monarch";
        auraColor = 'rgba(125, 95, 255, 0.5)';
        hornColor = '#7d5fff';
        eyeColor = '#00f0ff';
        bodyColor = '#1a0933';
      } else if (distance >= 100) {
        skinName = "Crimson Lord";
        auraColor = 'rgba(255, 69, 0, 0.4)';
        hornColor = '#ff4500';
        eyeColor = '#ffcc00';
        bodyColor = '#2b0900';
      }

      document.getElementById('skin-txt').innerText = skinName;
      document.getElementById('skin-txt').style.color = hornColor;

      // Dark Aura
      ctx.fillStyle = auraColor;
      ctx.beginPath();
      ctx.arc(x + 16, y + 24, 26, 0, Math.PI * 2);
      ctx.fill();

      // Cape / Body
      ctx.fillStyle = bodyColor;
      ctx.fillRect(x + 4, y + 14, 24, 28);

      // Inner Suit
      ctx.fillStyle = '#2d1b4e';
      ctx.fillRect(x + 8, y + 18, 16, 20);

      // Horns
      ctx.fillStyle = hornColor;
      ctx.beginPath();
      ctx.moveTo(x + 6, y + 10);
      ctx.lineTo(x + 2, y - 6);
      ctx.lineTo(x + 10, y + 6);
      ctx.fill();

      ctx.beginPath();
      ctx.moveTo(x + 26, y + 10);
      ctx.lineTo(x + 30, y - 6);
      ctx.lineTo(x + 22, y + 6);
      ctx.fill();

      // Glowing Eyes
      ctx.fillStyle = eyeColor;
      ctx.fillRect(x + 8, y + 10, 4, 3);
      ctx.fillRect(x + 20, y + 10, 4, 3);

      // Running Legs
      ctx.fillStyle = hornColor;
      let legOffset = moveForward ? Math.sin(animFrame * 0.3) * 6 : 0;
      ctx.fillRect(x + 6, y + 42, 6, 8 + legOffset);
      ctx.fillRect(x + 20, y + 42, 6, 8 - legOffset);
    }

    function draw() {
      // Background Sky
      let sky = ctx.createLinearGradient(0, 0, 0, canvas.height);
      sky.addColorStop(0, '#0d0722');
      sky.addColorStop(0.6, '#320e3b');
      sky.addColorStop(1, '#ff0055');
      ctx.fillStyle = sky;
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Sun
      ctx.fillStyle = '#ffcc00';
      ctx.beginPath();
      ctx.arc(420, 80, 40, 0, Math.PI * 2);
      ctx.fill();

      // Road
      ctx.fillStyle = '#0b0914';
      ctx.fillRect(0, 265, canvas.width, 75);

      // Neon Road Boundary
      ctx.fillStyle = '#00f0ff';
      ctx.fillRect(0, 265, canvas.width, 2);

      // Draw Shadows
      ctx.fillStyle = 'rgba(5, 3, 12, 0.9)';
      for (let s of shadows) {
        ctx.fillRect(s.x, s.y, s.width, s.height);
      }

      // Draw Health Orbs
      for (let o of orbs) {
        if (!o.collected) {
          ctx.fillStyle = '#00f0ff';
          ctx.beginPath();
          ctx.arc(o.x, o.y + Math.sin(animFrame * 0.1) * 3, o.radius, 0, Math.PI * 2);
          ctx.fill();
        }
      }

      // Draw Buildings
      for (let b of buildings) {
        ctx.fillStyle = '#181335';
        ctx.fillRect(b.x, b.y, b.width, b.height);
        ctx.strokeStyle = '#322863';
        ctx.strokeRect(b.x, b.y, b.width, b.height);

        // Windows
        ctx.fillStyle = '#ff0055';
        for (let wx = b.x + 10; wx < b.x + b.width - 12; wx += 16) {
          for (let wy = b.y + 12; wy < b.y + b.height - 15; wy += 22) {
            ctx.fillRect(wx, wy, 7, 10);
          }
        }
      }

      // Draw Demon
      drawDemon(demon.x, demon.y);
    }

    function loop() {
      update();
      draw();
      requestAnimationFrame(loop);
    }

    loop();
  </script>
</body>
</html>
