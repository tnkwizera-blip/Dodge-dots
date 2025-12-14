<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <title>Dodge the Dots</title>
  <style>
    html, body { margin:0; padding:0; height:100%; background:#0b0f14; overflow:hidden; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; }
    #ui {
      position: fixed; left: 12px; top: 12px; right: 12px; display:flex; justify-content:space-between; gap:12px;
      color:#e8eef7; font-weight:600; font-size:14px; pointer-events:none;
      text-shadow: 0 2px 8px rgba(0,0,0,.5);
    }
    #banner {
      position: fixed; left: 50%; top: 50%; transform: translate(-50%,-50%);
      color:#e8eef7; text-align:center; padding:16px 18px; border:1px solid rgba(255,255,255,.15);
      background: rgba(10,14,20,.85); border-radius: 14px; width:min(320px, 90vw);
      box-shadow: 0 10px 30px rgba(0,0,0,.45);
    }
    #banner h1 { margin:0 0 6px 0; font-size:18px; }
    #banner p { margin:6px 0; font-size:13px; opacity:.9; line-height:1.35; }
    #banner button {
      margin-top:10px; pointer-events:auto;
      border:0; border-radius: 10px; padding:10px 12px; font-weight:700;
      background:#2f7cff; color:white; width:100%;
    }
    canvas { display:block; width:100vw; height:100vh; touch-action:none; }
  </style>
</head>
<body>
  <div id="ui">
    <div id="score">Score: 0</div>
    <div id="best">Best: 0</div>
  </div>

  <div id="banner">
    <h1>Dodge the Dots</h1>
    <p>Drag to move the blue circle. Avoid the red dots.</p>
    <p>Survive to increase difficulty. Good luck.</p>
    <button id="startBtn">Start</button>
  </div>

  <canvas id="c"></canvas>

<script>
(() => {
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d");
  const scoreEl = document.getElementById("score");
  const bestEl = document.getElementById("best");
  const banner = document.getElementById("banner");
  const startBtn = document.getElementById("startBtn");

  function resize() {
    const dpr = Math.max(1, Math.min(2, window.devicePixelRatio || 1));
    canvas.width = Math.floor(innerWidth * dpr);
    canvas.height = Math.floor(innerHeight * dpr);
    ctx.setTransform(dpr,0,0,dpr,0,0);
  }
  addEventListener("resize", resize, { passive:true });
  resize();

  const bestKey = "dodge_best_v1";
  let best = Number(localStorage.getItem(bestKey) || 0);
  bestEl.textContent = `Best: ${best}`;

  const state = {
    running: false,
    t: 0,
    score: 0,
    difficulty: 1,
    lastSpawn: 0,
    spawnEvery: 0.9,
    dots: [],
    player: { x: innerWidth/2, y: innerHeight*0.72, r: 18, vx: 0, vy: 0, targetX: null, targetY: null },
  };

  function reset() {
    state.t = 0;
    state.score = 0;
    state.difficulty = 1;
    state.lastSpawn = 0;
    state.spawnEvery = 0.9;
    state.dots.length = 0;
    state.player.x = innerWidth/2;
    state.player.y = innerHeight*0.72;
    state.player.vx = 0;
    state.player.vy = 0;
    state.player.targetX = null;
    state.player.targetY = null;
    scoreEl.textContent = "Score: 0";
  }

  function rand(min, max){ return Math.random()*(max-min)+min; }

  function spawnDot() {
    // Spawn from a random edge
    const edge = Math.floor(Math.random()*4);
    let x, y, vx, vy;
    const speed = 80 + 18*state.difficulty + rand(0, 50);
    if (edge === 0) { x = rand(0, innerWidth); y = -12; }
    if (edge === 1) { x = innerWidth+12; y = rand(0, innerHeight); }
    if (edge === 2) { x = rand(0, innerWidth); y = innerHeight+12; }
    if (edge === 3) { x = -12; y = rand(0, innerHeight); }

    // Aim roughly toward player with some variance
    const dx = (state.player.x - x) + rand(-120, 120);
    const dy = (state.player.y - y) + rand(-120, 120);
    const mag = Math.hypot(dx, dy) || 1;
    vx = (dx / mag) * speed;
    vy = (dy / mag) * speed;

    state.dots.push({ x, y, r: rand(8, 14), vx, vy });
  }

  function collide(a, b) {
    const d = Math.hypot(a.x - b.x, a.y - b.y);
    return d < a.r + b.r;
  }

  let last = performance.now();
  function loop(now) {
    const dt = Math.min(0.033, (now - last)/1000);
    last = now;

    if (state.running) update(dt);
    render();

    requestAnimationFrame(loop);
  }

  function update(dt) {
    state.t += dt;
    state.score += dt * 10; // 10 pts/sec
    scoreEl.textContent = `Score: ${Math.floor(state.score)}`;

    // Increase difficulty gradually
    state.difficulty = 1 + state.t / 8;
    state.spawnEvery = Math.max(0.22, 0.9 - state.t / 18);

    // Spawn dots
    state.lastSpawn += dt;
    while (state.lastSpawn >= state.spawnEvery) {
      state.lastSpawn -= state.spawnEvery;
      spawnDot();
      if (state.difficulty > 3 && Math.random() < 0.25) spawnDot();
    }

    // Player movement toward target (smooth)
    const p = state.player;
    if (p.targetX != null && p.targetY != null) {
      const dx = p.targetX - p.x;
      const dy = p.targetY - p.y;
      p.vx += dx * 12 * dt;
      p.vy += dy * 12 * dt;
    }
    // damping
    p.vx *= Math.pow(0.0008, dt);
    p.vy *= Math.pow(0.0008, dt);

    p.x += p.vx * dt * 60;
    p.y += p.vy * dt * 60;

    // clamp to screen
    p.x = Math.max(p.r, Math.min(innerWidth - p.r, p.x));
    p.y = Math.max(p.r, Math.min(innerHeight - p.r, p.y));

    // Move dots and remove offscreen
    for (let i = state.dots.length - 1; i >= 0; i--) {
      const d = state.dots[i];
      d.x += d.vx * dt;
      d.y += d.vy * dt;
      if (d.x < -80 || d.x > innerWidth + 80 || d.y < -80 || d.y > innerHeight + 80) {
        state.dots.splice(i, 1);
        continue;
      }
      if (collide(p, d)) {
        gameOver();
        return;
      }
    }
  }

  function gameOver() {
    state.running = false;
    const s = Math.floor(state.score);
    if (s > best) {
      best = s;
      localStorage.setItem(bestKey, String(best));
      bestEl.textContent = `Best: ${best}`;
    }
    banner.querySelector("h1").textContent = "Game Over";
    banner.querySelector("p").textContent = `Score: ${s}  |  Best: ${best}`;
    banner.querySelectorAll("p")[1].textContent = "Tap Start to try again.";
    startBtn.textContent = "Start";
    banner.style.display = "block";
  }

  function render() {
    ctx.clearRect(0,0,innerWidth,innerHeight);

    // background grid glow
    ctx.save();
    ctx.globalAlpha = 0.25;
    const step = 40;
    for (let x=0; x<innerWidth; x+=step) {
      ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,innerHeight);
      ctx.strokeStyle = "rgba(255,255,255,0.04)";
      ctx.stroke();
    }
    for (let y=0; y<innerHeight; y+=step) {
      ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(innerWidth,y);
      ctx.strokeStyle = "rgba(255,255,255,0.04)";
      ctx.stroke();
    }
    ctx.restore();

    // dots
    for (const d of state.dots) {
      ctx.beginPath();
      ctx.arc(d.x, d.y, d.r, 0, Math.PI*2);
      ctx.fillStyle = "#ff3b3b";
      ctx.fill();
    }

    // player
    const p = state.player;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
    ctx.fillStyle = "#2f7cff";
    ctx.fill();

    // soft highlight
    ctx.beginPath();
    ctx.arc(p.x - p.r*0.35, p.y - p.r*0.35, p.r*0.35, 0, Math.PI*2);
    ctx.fillStyle = "rgba(255,255,255,0.22)";
    ctx.fill();
  }

  function setTarget(clientX, clientY) {
    state.player.targetX = clientX;
    state.player.targetY = clientY;
  }

  // Touch + mouse controls
  let dragging = false;
  canvas.addEventListener("pointerdown", (e) => {
    dragging = true;
    canvas.setPointerCapture(e.pointerId);
    setTarget(e.clientX, e.clientY);
  }, { passive:false });

  canvas.addEventListener("pointermove", (e) => {
    if (!dragging) return;
    setTarget(e.clientX, e.clientY);
  }, { passive:false });

  canvas.addEventListener("pointerup", () => { dragging = false; }, { passive:true });
  canvas.addEventListener("pointercancel", () => { dragging = false; }, { passive:true });

  startBtn.addEventListener("click", () => {
    reset();
    banner.style.display = "none";
    state.running = true;
  });

  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
