<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Honigjäger — Schieß die Bienen!</title>
  <style>
    html,body{height:100%;margin:0;background:linear-gradient(#bde0fe,#e0f7fa);font-family:Inter,Segoe UI,Roboto,Arial}
    #game-wrap{display:flex;flex-direction:column;align-items:center;padding:12px;gap:8px}
    canvas{background:linear-gradient(#89c2ff,#d6f5ff);border-radius:12px;box-shadow:0 8px 30px rgba(0,0,0,0.15)}
    .hud{display:flex;gap:12px;align-items:center}
    .btn{padding:6px 10px;border-radius:8px;background:#ffd166;border:none;cursor:pointer;font-weight:600}
    .info{background:rgba(255,255,255,0.6);padding:8px 12px;border-radius:8px}
    @media(min-width:900px){canvas{width:900px;height:600px}}    
  </style>
</head>
<body>
  <div id="game-wrap">
    <h1>Honigjäger 🍯🐝</h1>
    <div class="hud">
      <div class="info">Punkte: <span id="score">0</span></div>
      <div class="info">Treffer: <span id="hits">0</span></div>
      <button id="restart" class="btn">Neustarten</button>
      <div style="margin-left:12px;color:#444;font-size:14px">Klicke, um Honig zu schießen. Halte gedrückt für Dauerfeuer (Cooldown)</div>
    </div>
    <canvas id="c" width="800" height="500"></canvas>
    <div style="max-width:900px;text-align:center;color:#333;opacity:0.9">
      Ziele die Bienen mit Honigkugeln. Manche Bienen sind schneller oder brauchen mehrere Treffer.
    </div>
  </div>

<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  const scoreEl = document.getElementById('score');
  const hitsEl = document.getElementById('hits');
  const restartBtn = document.getElementById('restart');

  let W = canvas.width, H = canvas.height;
  function resize() {
    // keep internal resolution fixed; canvas displayed size can be responsive by CSS
    W = canvas.width = Math.max(600, Math.min(1200, Math.floor(window.innerWidth * 0.9)));
    H = canvas.height = Math.floor(W * 0.625);
  }
  resize();
  window.addEventListener('resize', resize);

  // game state
  let bees = [];
  let bullets = [];
  let particles = [];
  let score = 0;
  let hits = 0;
  let mouseDown = false;
  let lastShot = 0;
  const SHOT_COOLDOWN = 180; // ms
  let lastTime = performance.now();

  function rand(min,max){return Math.random()*(max-min)+min}

  // Bee class
  class Bee{
    constructor(x,y,level=1){
      this.x=x;this.y=y;this.vx=rand(-1.2,1.2);this.vy=rand(-0.7,0.7);
      this.r = 18 + level*4;
      this.level = level; // hits to die
      this.hp = level;
      this.angle = 0;
      this.wiggle = rand(0,Math.PI*2);
    }
    update(dt){
      // basic wandering + attraction to center
      this.wiggle += dt*0.01;
      this.x += this.vx + Math.sin(this.wiggle)*0.6;
      this.y += this.vy + Math.cos(this.wiggle)*0.3;
      this.angle += (this.vx)*0.05;
      // bounce
      if(this.x < this.r) {this.x = this.r; this.vx *= -1}
      if(this.x > W-this.r) {this.x = W-this.r; this.vx *= -1}
      if(this.y < this.r) {this.y = this.r; this.vy *= -1}
      if(this.y > H-this.r) {this.y = H-this.r; this.vy *= -1}
    }
    draw(ctx){
      ctx.save();
      ctx.translate(this.x,this.y);
      ctx.rotate(Math.sin(this.wiggle)*0.2);
      // body
      ctx.beginPath();
      ctx.ellipse(0,0,this.r, this.r*0.65, 0, 0, Math.PI*2);
      ctx.fillStyle = '#f6c33c'; ctx.fill();
      // stripes
      ctx.fillStyle = '#3b3b3b';
      ctx.fillRect(-this.r*0.7, -this.r*0.25, this.r*1.4, this.r*0.17);
      ctx.fillRect(-this.r*0.4, -this.r*0.02, this.r*0.8, this.r*0.17);
      // wings
      ctx.globalAlpha = 0.85;
      ctx.beginPath(); ctx.ellipse(-this.r*0.25, -this.r*0.9, this.r*0.6, this.r*0.35, -0.4, 0, Math.PI*2); ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.fill();
      ctx.beginPath(); ctx.ellipse(this.r*0.25, -this.r*0.9, this.r*0.6, this.r*0.35, 0.4, 0, Math.PI*2); ctx.fill();
      ctx.globalAlpha = 1;
      // eyes
      ctx.fillStyle='#111'; ctx.beginPath(); ctx.arc(this.r*0.45, -this.r*0.05, this.r*0.12,0,Math.PI*2); ctx.fill();
      // hp indicator
      if(this.level>1){
        ctx.fillStyle='#fff'; ctx.font='12px sans-serif'; ctx.textAlign='center'; ctx.fillText(this.hp+'/'+this.level,0,this.r+14);
      }
      ctx.restore();
    }
  }

  // bullet
  class Bullet{
    constructor(x,y,dx,dy){this.x=x;this.y=y;this.vx=dx;this.vy=dy;this.r=7}
    update(dt){this.x += this.vx*dt*0.06; this.y += this.vy*dt*0.06}
    draw(ctx){ctx.beginPath(); ctx.arc(this.x,this.y,this.r,0,Math.PI*2); ctx.fillStyle='#d97706'; ctx.fill(); ctx.strokeStyle='rgba(0,0,0,0.15)'; ctx.stroke()}
  }

  // particles for hit effect
  class Particle{constructor(x,y,vx,vy,life){this.x=x;this.y=y;this.vx=vx;this.vy=vy;this.life=life;this.max=life}
    update(dt){this.x+=this.vx*dt*0.06;this.y+=this.vy*dt*0.06;this.life-=dt*0.06}
    draw(ctx){if(this.life<=0) return;ctx.globalAlpha=Math.max(0,this.life/this.max);ctx.beginPath();ctx.arc(this.x,this.y,3,0,Math.PI*2);ctx.fillStyle='#ffb547';ctx.fill();ctx.globalAlpha=1}
  }

  function spawnBee(){
    const edge = Math.random();
    let x,y;
    if(edge<0.25){x=0;y=rand(0,H)}
    else if(edge<0.5){x=W;y=rand(0,H)}
    else if(edge<0.75){x=rand(0,W);y=0}
    else {x=rand(0,W);y=H}
    // level chance
    const r = Math.random();
    let level = r<0.75?1: r<0.95?2:3;
    bees.push(new Bee(x,y,level));
  }

  // initial bees
  for(let i=0;i<5;i++) spawnBee();

  function shoot(targetX,targetY){
    const x = W/2; const y = H; // shooter at bottom-center
    const dx = targetX - x; const dy = targetY - y; const dist = Math.hypot(dx,dy)||1;
    const speed = 9;
    bullets.push(new Bullet(x,y, dx/dist*speed, dy/dist*speed));
    // small recoil particles
    for(let i=0;i<6;i++) particles.push(new Particle(x,y, rand(-1,1)*3, rand(-2,0)*3, 40 ));
    // sound
    playShootSound();
  }

  // collision
  function checkCollisions(){
    for(let bI=bees.length-1;bI>=0;bI--){
      const bee = bees[bI];
      for(let i=bullets.length-1;i>=0;i--){
        const bul = bullets[i];
        const dx = bee.x - bul.x; const dy = bee.y - bul.y; const d2 = dx*dx+dy*dy;
        const rsum = bee.r + bul.r;
        if(d2 < rsum*rsum){
          // hit
          bullets.splice(i,1);
          bee.hp -= 1; hits++; hitsEl.textContent = hits;
          // spawn particles
          for(let p=0;p<12;p++) particles.push(new Particle(bul.x,bul.y, rand(-3,3), rand(-3,3), 60 ));
          if(bee.hp<=0){
            score += bee.level * 10;
            scoreEl.textContent = score;
            // explosion
            for(let p=0;p<24;p++) particles.push(new Particle(bee.x,bee.y, rand(-4,4), rand(-4,4), 80 ));
            bees.splice(bI,1);
          } else {
            score += 5; scoreEl.textContent = score;
          }
          break;
        }
      }
    }
  }

  // draw shooter
  function drawShooter(ctx){
    const x = W/2; const y = H - 24;
    ctx.save();
    ctx.translate(x,y);
    // platform
    ctx.fillStyle='#6b705c'; ctx.beginPath(); ctx.roundRect(-40,-8,80,16,8); ctx.fill();
    // jar
    ctx.beginPath(); ctx.ellipse(0,-36,18,20,0,0,Math.PI*2); ctx.fillStyle='#e9c46a'; ctx.fill(); ctx.strokeStyle='rgba(0,0,0,0.15)'; ctx.stroke();
    ctx.restore();
  }

  // main loop
  function loop(now){
    const dt = now - lastTime; lastTime = now;
    // spawn logic
    if(Math.random() < 0.01 + Math.min(0.06, score*0.0006)){
      spawnBee();
    }

    // update
    bees.forEach(b => b.update(dt));
    bullets.forEach(b => b.update(dt));
    particles.forEach(p => p.update(dt));
    // remove offscreen bullets
    for(let i=bullets.length-1;i>=0;i--) if(bullets[i].x< -50 || bullets[i].x>W+50 || bullets[i].y<-50 || bullets[i].y>H+50) bullets.splice(i,1);
    for(let i=particles.length-1;i>=0;i--) if(particles[i].life<=0) particles.splice(i,1);

    checkCollisions();

    // draw
    ctx.clearRect(0,0,W,H);
    // background flowers / grass
    for(let g=0; g<35; g++){
      const gx = (g*73) % W; const gy = H - 20 - (Math.sin(g*0.9+now*0.001)*6);
      ctx.fillStyle='rgba(34,139,34,0.08)'; ctx.fillRect(gx,gy,20,40);
    }

    bees.forEach(b => b.draw(ctx));
    bullets.forEach(b => b.draw(ctx));
    particles.forEach(p => p.draw(ctx));
    drawShooter(ctx);

    requestAnimationFrame(loop);
  }

  // input
  canvas.addEventListener('mousedown', e => { mouseDown = true; attemptShoot(e); });
  canvas.addEventListener('mouseup', e => { mouseDown = false; });
  canvas.addEventListener('mousemove', e => { /* could show aim */ });
  canvas.addEventListener('touchstart', e => { mouseDown = true; attemptShoot(e.touches[0]); e.preventDefault(); }, {passive:false});
  canvas.addEventListener('touchend', e => { mouseDown = false; }, {passive:false});

  function getCanvasPos(e){
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX - rect.left) * (canvas.width / rect.width);
    const y = (e.clientY - rect.top) * (canvas.height / rect.height);
    return {x,y};
  }

  function attemptShoot(e){
    const now = performance.now();
    if(now - lastShot < SHOT_COOLDOWN) return;
    lastShot = now;
    const p = getCanvasPos(e);
    shoot(p.x,p.y);
  }

  // auto-fire while held
  setInterval(()=>{
    if(mouseDown) {
      // try to shoot at current mouse position (closest touch/mouse)
      // We'll query last mouse position from window event
      // store last pos
      if(window._lastPointer) attemptShoot(window._lastPointer);
    }
  }, 120);

  // track pointer
  window.addEventListener('pointermove', e => { window._lastPointer = e; });

  restartBtn.addEventListener('click', () => { bees=[]; bullets=[]; particles=[]; score=0; hits=0; scoreEl.textContent='0'; hitsEl.textContent='0'; for(let i=0;i<5;i++) spawnBee(); });

  // audio: tiny shoot and hit sounds using WebAudio
  const AudioCtx = window.AudioContext || window.webkitAudioContext;
  let audioC = null;
  function ensureAudio(){ if(!audioC) audioC = new AudioCtx(); }
  function playShootSound(){ try{ ensureAudio(); const o=audioC.createOscillator(); const g=audioC.createGain(); o.type='sine'; o.frequency.value=720; g.gain.value=0.06; o.connect(g); g.connect(audioC.destination); o.start(); o.stop(audioC.currentTime + 0.06);}catch(e){} }

  // start
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
