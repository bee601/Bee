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
    .upgrade{padding:6px 10px;border-radius:8px;background:#a7c957;border:none;cursor:pointer;font-weight:600;transition:0.2s}
    .upgrade:hover{background:#74a329}
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
      <button id="upgradeUltra" class="upgrade">Upgrade: Ultra Feuer (Kosten: 5000)</button>
      <button id="upgradeGiga" class="upgrade">Upgrade: GIGA Honigstrahl (Kosten: 20000)</button>
      <button id="upgradePoints" class="upgrade">Upgrade: Mehr Punkte pro Kill (Kosten: 550)</button>
      <div style="margin-left:12px;color:#444;font-size:14px;user-select:none">Bewege die Maus, um zu zielen, halte gedrückt zum Dauerfeuer und verschiebe die Plattform!</div>
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
  const upgradeUltraBtn = document.getElementById('upgradeUltra');
  const upgradeGigaBtn = document.getElementById('upgradeGiga');
  const upgradePointsBtn = document.getElementById('upgradePoints');

  let W = canvas.width, H = canvas.height;
  function resize() { W = canvas.width = Math.max(600, Math.min(1200, Math.floor(window.innerWidth * 0.9))); H = canvas.height = Math.floor(W * 0.625); }
  resize(); window.addEventListener('resize', resize);

  let bees = [], bullets = [], particles = [];
  let score = 0, hits = 0;
  let mouseDown = false;
  let lastShot = 0;
  let SHOT_COOLDOWN = 100; // ms
  let bulletDamage = 1;
  let multiShot = false;
  let shooterX = W / 2;
  let shooterTargetX = W / 2;

  let pointsPerKill = 10;
  let spawnLimit = 20;
  let spawnInterval = 60; // kürzere Spawnzeit für schnellere Bienen

  function rand(min,max){return Math.random()*(max-min)+min}

  class Bee{
    constructor(x,y,level=1){this.x=x; this.y=y; this.vx=rand(-2,2); this.vy=rand(-1.2,1.2); this.r=18+level*6; this.level=level; this.hp=level*2+1; this.wiggle=rand(0,Math.PI*2);}
    update(){this.wiggle+=0.05; this.x+=this.vx+Math.sin(this.wiggle)*2; this.y+=this.vy+Math.cos(this.wiggle)*1.2; if(this.x<this.r){this.x=this.r; this.vx*=-1} if(this.x>W-this.r){this.x=W-this.r; this.vx*=-1} if(this.y<this.r){this.y=this.r; this.vy*=-1} if(this.y>H-this.r){this.y=H-this.r; this.vy*=-1}}
    draw(ctx){ ctx.save(); ctx.translate(this.x,this.y); ctx.beginPath(); ctx.ellipse(0,0,this.r,this.r*0.65,0,0,Math.PI*2); ctx.fillStyle='#f6c33c'; ctx.fill(); ctx.fillStyle='#3b3b3b'; ctx.fillRect(-this.r*0.7,-this.r*0.25,this.r*1.4,this.r*0.17); ctx.fillRect(-this.r*0.4,-this.r*0.02,this.r*0.8,this.r*0.17); ctx.globalAlpha=0.85; ctx.beginPath(); ctx.ellipse(-this.r*0.25,-this.r*0.9,this.r*0.6,this.r*0.35,-0.4,0,Math.PI*2); ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.fill(); ctx.beginPath(); ctx.ellipse(this.r*0.25,-this.r*0.9,this.r*0.6,this.r*0.35,0.4,0,Math.PI*2); ctx.fill(); ctx.globalAlpha=1; ctx.fillStyle='#111'; ctx.beginPath(); ctx.arc(this.r*0.45,-this.r*0.05,this.r*0.12,0,Math.PI*2); ctx.fill(); if(this.hp>1){ctx.fillStyle='#fff'; ctx.font='12px sans-serif'; ctx.textAlign='center'; ctx.fillText(this.hp+'/'+(this.level*2+1),0,this.r+14);} ctx.restore();}
  }

  class Bullet{constructor(x,y,dx,dy){this.x=x;this.y=y;this.vx=dx*12;this.vy=dy*12;this.r=7} update(){this.x+=this.vx;this.y+=this.vy} draw(ctx){ctx.beginPath();ctx.arc(this.x,this.y,this.r,0,Math.PI*2);ctx.fillStyle='#d97706';ctx.fill()}}
  class Particle{constructor(x,y,vx,vy,life){this.x=x;this.y=y;this.vx=vx;this.vy=vy;this.life=life;this.max=life} update(){this.x+=this.vx;this.y+=this.vy;this.life-=1} draw(ctx){if(this.life<=0)return;ctx.globalAlpha=this.life/this.max;ctx.beginPath();ctx.arc(this.x,this.y,3,0,Math.PI*2);ctx.fillStyle='#ffb547';ctx.fill();ctx.globalAlpha=1}}

  function spawnBee(){ const x=rand(40,W-40); const y=rand(20,Math.max(60,H/7)); const r=Math.random(); const level=r<0.7?1:r<0.92?2:3; bees.push(new Bee(x,y,level)); }
  for(let i=0;i<5;i++) spawnBee();

  function shoot(tx,ty){ const dx=tx-shooterX, dy=ty-(H-40), dist=Math.hypot(dx,dy)||1; bullets.push(new Bullet(shooterX,H-40,dx/dist,dy/dist)); if(multiShot){ bullets.push(new Bullet(shooterX,H-40,dx/dist*0.9-dy/dist*0.3,dy/dist*0.9+dx/dist*0.3)); bullets.push(new Bullet(shooterX,H-40,dx/dist*0.9+dy/dist*0.3,dy/dist*0.9-dx/dist*0.3));}}

  function checkCollisions(){
    for(let bI=bees.length-1;bI>=0;bI--){ const bee=bees[bI]; for(let i=bullets.length-1;i>=0;i--){ const bul=bullets[i]; const dx=bee.x-bul.x,dy=bee.y-bul.y; if(dx*dx+dy*dy<(bee.r+bul.r)*(bee.r+bul.r)){ bullets.splice(i,1); bee.hp-=bulletDamage; hits++; hitsEl.textContent=hits;
      if(hits % 100 === 0){ spawnLimit+=5; } // nach jedem 100. Treffer mehr Bienen spawnen
      if(bee.hp<=0){ score+=bee.level*pointsPerKill; scoreEl.textContent=score; for(let p=0;p<18;p++) particles.push(new Particle(bee.x,bee.y,rand(-3,3),rand(-3,3),30)); bees.splice(bI,1); } else { score+=Math.max(1,Math.floor(pointsPerKill/3)); scoreEl.textContent=score; } break; } } }
  }

  function drawShooter(ctx){ const y=H-24; shooterX+=(shooterTargetX-shooterX)*0.15; ctx.save(); ctx.translate(shooterX,y); ctx.fillStyle='#6b705c'; ctx.beginPath(); ctx.rect(-40,-8,80,16); ctx.fill(); ctx.beginPath(); ctx.ellipse(0,-36,18,20,0,0,Math.PI*2); ctx.fillStyle='#e9c46a'; ctx.fill(); ctx.restore(); }

  let spawnTimer=0;

  function loop(){
    if(mouseDown) attemptShoot({clientX:lastPointer.x,clientY:lastPointer.y});
    ctx.clearRect(0,0,W,H);
    spawnTimer++;
    if(spawnTimer>=spawnInterval){ spawnTimer=0; if(bees.length<spawnLimit) spawnBee(); } // schnellere Spawns
    bees.forEach(b=>b.update()); bullets.forEach(b=>b.update()); particles.forEach(p=>p.update()); checkCollisions();
    ctx.clearRect(0,0,W,H);
    for(let g=0;g<30;g++){ const gx=(g*123)%W, gy=H-18-(Math.sin(g*0.9+performance.now()*0.001)*6); ctx.fillStyle='rgba(34,139,34,0.04)'; ctx.fillRect(gx,gy,20,40); }
    bees.forEach(b=>b.draw(ctx)); bullets.forEach(b=>b.draw(ctx)); particles.forEach(p=>p.draw(ctx)); drawShooter(ctx);
    requestAnimationFrame(loop);
  }

  let lastPointer={x:W/2,y:H/2};
  canvas.addEventListener('mousemove', e => { lastPointer=getCanvasPos(e); shooterTargetX=lastPointer.x; });
  canvas.addEventListener('mousedown', e => { mouseDown=true; e.preventDefault(); lastPointer=getCanvasPos(e); attemptShoot(e); });
  window.addEventListener('mouseup', e => { mouseDown=false; });
  canvas.addEventListener('touchstart', e => { mouseDown=true; e.preventDefault(); lastPointer=getCanvasPos(e.touches[0]); attemptShoot(e.touches[0]);},{passive:false});
  canvas.addEventListener('touchend', e => { mouseDown=false; });
  function getCanvasPos(e){ const rect=canvas.getBoundingClientRect(); return {x:(e.clientX-rect.left)*(canvas.width/rect.width),y:(e.clientY-rect.top)*(canvas.height/rect.height)}; }
  function attemptShoot(e){ const now=performance.now(); if(now-lastShot<SHOT_COOLDOWN)return; lastShot=now; const p=getCanvasPos(e); shoot(p.x,p.y); }

  // Upgrades
  let upgradeLevels={speed:0,damage:0,multi:0,ultra:0,giga:0,points:0};
  const upgradeCosts={speed:[100,200,400],damage:[150,300,600],multi:[1000,2500,5000],ultra:[5000,12000],giga:[20000,50000],points:[550,1200,2500]};
  const upgradeButtons={speed:upgradeSpeedBtn,damage:upgradeDamageBtn,multi:upgradeMultiBtn,ultra:upgradeUltraBtn,giga:upgradeGigaBtn,points:upgradePointsBtn};

  function buyUpgrade(type){
    const lvl=upgradeLevels[type];
    if(lvl>=upgradeCosts[type].length) return;
    const cost=upgradeCosts[type][lvl];
    if(score>=cost){
      score-=cost; scoreEl.textContent=score;
      upgradeLevels[type]++;
      if(type==='speed') SHOT_COOLDOWN=Math.max(5, SHOT_COOLDOWN-15);
      if(type==='damage') bulletDamage+=1;
      if(type==='multi') multiShot=true;
      if(type==='ultra') {SHOT_COOLDOWN=Math.max(5,SHOT_COOLDOWN-10); bulletDamage+=2;}
      if(type==='giga'){SHOT_COOLDOWN=5; bulletDamage+=5; multiShot=true;}
      if(type==='points') pointsPerKill+=5;
      if(type==='points' || type==='speed') spawnLimit+=5;
      const btn=upgradeButtons[type];
      const nextCost=upgradeCosts[type][upgradeLevels[type]];
      if(nextCost) btn.textContent=`Upgrade: ${btn.textContent.split('(')[0]} (Kosten: ${nextCost})`;
      else btn.textContent=`Upgrade: ${btn.textContent.split('(')[0]} ✅`; btn.disabled=!nextCost;
    }
  }

  Object.keys(upgradeButtons).forEach(type=>{ upgradeButtons[type].addEventListener('click',()=>buyUpgrade(type)); });

  restartBtn.addEventListener('click',()=>{ bees=[]; bullets=[]; particles=[]; score=0; hits=0; scoreEl.textContent='0'; hitsEl.textContent='0'; upgradeLevels={speed:0,damage:0,multi:0,ultra:0,giga:0,points:0}; for(let i=0;i<5;i++) spawnBee(); });

  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
