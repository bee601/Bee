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
    .hud{display:flex;gap:12px;align-items:center;flex-wrap:wrap;justify-content:center}
    .btn{padding:6px 10px;border-radius:8px;background:#ffd166;border:none;cursor:pointer;font-weight:600}
    .info{background:rgba(255,255,255,0.6);padding:8px 12px;border-radius:8px}
    .upgrade{padding:6px 10px;border-radius:8px;background:#a7c957;border:none;cursor:pointer;font-weight:600;transition:0.2s;}
    .upgrade:hover{background:#74a329;}
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
      <button id="upgradeSpeed" class="upgrade">Upgrade: Schnellere Schüsse (Kosten: 100)</button>
      <button id="upgradeDamage" class="upgrade">Upgrade: Stärkerer Honig (Kosten: 150)</button>
      <button id="upgradeMulti" class="upgrade">Upgrade: Mehrfachschuss (Kosten: 1000)</button>
      <div style="margin-left:12px;color:#444;font-size:14px">Bewege die Maus, um zu zielen, und klicke, um Honig zu schießen!</div>
    </div>
    <canvas id="c" width="800" height="500"></canvas>
  </div>

<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  const scoreEl = document.getElementById('score');
  const hitsEl = document.getElementById('hits');
  const restartBtn = document.getElementById('restart');
  const upgradeSpeedBtn = document.getElementById('upgradeSpeed');
  const upgradeDamageBtn = document.getElementById('upgradeDamage');
  const upgradeMultiBtn = document.getElementById('upgradeMulti');

  let W = canvas.width, H = canvas.height;
  function resize() {
    W = canvas.width = Math.max(600, Math.min(1200, Math.floor(window.innerWidth * 0.9)));
    H = canvas.height = Math.floor(W * 0.625);
  }
  resize();
  window.addEventListener('resize', resize);

  let bees = [], bullets = [], particles = [];
  let score = 0, hits = 0;
  let mouseDown = false;
  let lastShot = 0;
  let SHOT_COOLDOWN = 100;
  let bulletDamage = 1;
  let multiShot = false;
  let shooterX = W / 2;

  function rand(min,max){return Math.random()*(max-min)+min}

  class Bee{
    constructor(x,y,level=1){
      this.x=x;this.y=y;this.vx=rand(-2,2);this.vy=rand(-1.5,1.5);
      this.r = 18 + level*4;
      this.level = level;
      this.hp = level;
      this.wiggle = rand(0,Math.PI*2);
    }
    update(){
      this.wiggle += 0.05;
      this.x += this.vx + Math.sin(this.wiggle)*2;
      this.y += this.vy + Math.cos(this.wiggle)*1.5;
      if(this.x < this.r) {this.x = this.r; this.vx *= -1}
      if(this.x > W-this.r) {this.x = W-this.r; this.vx *= -1}
      if(this.y < this.r) {this.y = this.r; this.vy *= -1}
      if(this.y > H-this.r) {this.y = H-this.r; this.vy *= -1}
    }
    draw(ctx){
      ctx.save();
      ctx.translate(this.x,this.y);
      ctx.beginPath();
      ctx.ellipse(0,0,this.r, this.r*0.65, 0, 0, Math.PI*2);
      ctx.fillStyle = '#f6c33c'; ctx.fill();
      ctx.fillStyle = '#3b3b3b';
      ctx.fillRect(-this.r*0.7, -this.r*0.25, this.r*1.4, this.r*0.17);
      ctx.fillRect(-this.r*0.4, -this.r*0.02, this.r*0.8, this.r*0.17);
      ctx.globalAlpha = 0.85;
      ctx.beginPath(); ctx.ellipse(-this.r*0.25, -this.r*0.9, this.r*0.6, this.r*0.35, -0.4, 0, Math.PI*2); ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.fill();
      ctx.beginPath(); ctx.ellipse(this.r*0.25, -this.r*0.9, this.r*0.6, this.r*0.35, 0.4, 0, Math.PI*2); ctx.fill();
      ctx.globalAlpha = 1;
      ctx.fillStyle='#111'; ctx.beginPath(); ctx.arc(this.r*0.45, -this.r*0.05, this.r*0.12,0,Math.PI*2); ctx.fill();
      ctx.restore();
    }
  }

  class Bullet{
    constructor(x,y,dx,dy){this.x=x;this.y=y;this.vx=dx*10;this.vy=dy*10;this.r=7}
    update(){this.x += this.vx; this.y += this.vy;}
    draw(ctx){ctx.beginPath(); ctx.arc(this.x,this.y,this.r,0,Math.PI*2); ctx.fillStyle='#d97706'; ctx.fill();}
  }

  class Particle{constructor(x,y,vx,vy,life){this.x=x;this.y=y;this.vx=vx;this.vy=vy;this.life=life;this.max=life}
    update(){this.x+=this.vx;this.y+=this.vy;this.life-=1}
    draw(ctx){if(this.life<=0)return;ctx.globalAlpha=this.life/this.max;ctx.beginPath();ctx.arc(this.x,this.y,3,0,Math.PI*2);ctx.fillStyle='#ffb547';ctx.fill();ctx.globalAlpha=1}
  }

  function spawnBee(){
    const x=rand(0,W),y=rand(0,H/2);
    const level=Math.random()<0.8?1:2;
    bees.push(new Bee(x,y,level));
  }

  for(let i=0;i<5;i++)spawnBee();

  function shoot(targetX,targetY){
    const x=shooterX,y=H-40;
    const dx=targetX-x,dy=targetY-y,dist=Math.hypot(dx,dy)||1;
    const vx=dx/dist,vy=dy/dist;
    bullets.push(new Bullet(x,y,vx,vy));
    if(multiShot){
      bullets.push(new Bullet(x,y,vx*0.9-vy*0.3,vy*0.9+vx*0.3));
      bullets.push(new Bullet(x,y,vx*0.9+vy*0.3,vy*0.9-vx*0.3));
    }
  }

  function checkCollisions(){
    for(let bI=bees.length-1;bI>=0;bI--){
      const bee=bees[bI];
      for(let i=bullets.length-1;i>=0;i--){
        const bul=bullets[i];
        const dx=bee.x-bul.x,dy=bee.y-bul.y;
        if(dx*dx+dy*dy<(bee.r+bul.r)**2){
          bullets.splice(i,1);
          bee.hp-=bulletDamage;
          hits++;hitsEl.textContent=hits;
          if(bee.hp<=0){
            score+=bee.level*10;
            for(let p=0;p<15;p++)particles.push(new Particle(bee.x,bee.y,rand(-2,2),rand(-2,2),30));
            bees.splice(bI,1);
          }else score+=5;
          scoreEl.textContent=score;
          break;
        }
      }
    }
  }

  function drawShooter(ctx){
    const y=H-24;
    ctx.save();
    ctx.translate(shooterX,y);
    ctx.fillStyle='#6b705c';
    ctx.beginPath();
    ctx.rect(-40,-8,80,16);
    ctx.fill();
    ctx.beginPath();
    ctx.ellipse(0,-36,18,20,0,0,Math.PI*2);
    ctx.fillStyle='#e9c46a';
    ctx.fill();
    ctx.restore();
  }

  let spawnTimer=0;

  function loop(){
    ctx.clearRect(0,0,W,H);
    spawnTimer++;
    if(spawnTimer>=60){
      spawnTimer=0;
      if(bees.length<20){spawnBee();}
    }

    bees.forEach(b=>b.update());
    bullets.forEach(b=>b.update());
    particles.forEach(p=>p.update());
    checkCollisions();

    bees.forEach(b=>b.draw(ctx));
    bullets.forEach(b=>b.draw(ctx));
    particles.forEach(p=>p.draw(ctx));
    drawShooter(ctx);

    requestAnimationFrame(loop);
  }

  let lastPointer = {x: W / 2, y: H / 2};

  canvas.addEventListener('mousemove', e => {
    const p = getCanvasPos(e);
    shooterX = p.x; // Spieler bewegt den Schützen
    lastPointer = p;
  });

  canvas.addEventListener('mousedown', e => {
    mouseDown = true;
    lastPointer = getCanvasPos(e);
    attemptShoot(e);
  });
  canvas.addEventListener('mouseup', () => mouseDown = false);

  setInterval(() => {
    if (mouseDown) attemptShoot({clientX: lastPointer.x, clientY: lastPointer.y});
  }, SHOT_COOLDOWN);

  function getCanvasPos(e){const rect=canvas.getBoundingClientRect();return{x:(e.clientX-rect.left)*(canvas.width/rect.width),y:(e.clientY-rect.top)*(canvas.height/rect.height)}}

  function attemptShoot(e){
    const now = performance.now();
    if (now - lastShot < SHOT_COOLDOWN) return;
    lastShot = now;
    const p = getCanvasPos(e);
    shoot(p.x, p.y);
  }

  restartBtn.addEventListener('click',()=>{bees=[];bullets=[];particles=[];score=0;hits=0;scoreEl.textContent='0';hitsEl.textContent='0';for(let i=0;i<5;i++)spawnBee();});

  upgradeSpeedBtn.addEventListener('click',()=>{
    if(score>=100){score-=100;SHOT_COOLDOWN=Math.max(40,SHOT_COOLDOWN-20);scoreEl.textContent=score;upgradeSpeedBtn.textContent='Upgrade: Schnellere Schüsse ✅';}
  });

  upgradeDamageBtn.addEventListener('click',()=>{
    if(score>=150){score-=150;bulletDamage+=1;scoreEl.textContent=score;upgradeDamageBtn.textContent='Upgrade: Stärkerer Honig ✅';}
  });

  upgradeMultiBtn.addEventListener('click',()=>{
    if(score>=1000){score-=1000;multiShot=true;scoreEl.textContent=score;upgradeMultiBtn.textContent='Upgrade: Mehrfachschuss ✅';}
  });

  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
