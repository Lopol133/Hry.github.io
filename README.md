<!doctype html>
<html lang="cs">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <title>Neon Skokánci</title>
  <style>
    :root{ --bg:#04060a; --neon:#00f0ff; --accent:#ff00f0; --glass:rgba(255,255,255,0.03)}
    html,body{height:100%;margin:0;background:linear-gradient(180deg,#02040a 0%, #061226 100%);font-family:Inter,system-ui,Segoe UI,Roboto,Arial}
    .app{height:100vh;display:flex;align-items:center;justify-content:center}
    .game-shell{width:100%;max-width:1366px;height:70vh;max-height:720px;background:radial-gradient(ellipse at 20% 10%, rgba(0,240,255,0.04), transparent),linear-gradient(180deg, rgba(255,255,255,0.02), rgba(0,0,0,0.08));border-radius:18px;box-shadow:0 10px 40px rgba(0,0,0,0.6);position:relative;overflow:hidden}
    .grid{position:absolute;inset:0;background-image:linear-gradient(transparent 90%, rgba(255,255,255,0.02) 100%),linear-gradient(90deg, transparent 90%, rgba(255,255,255,0.02) 100%);background-size:120px 120px}
    .menu{position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:18px;z-index:20}
    .title{font-size:48px;letter-spacing:2px;color:var(--neon);text-shadow:0 0 8px rgba(0,240,255,0.7),0 0 24px rgba(0,240,255,0.2);font-weight:800}
    .menu-row{display:flex;gap:12px}
    .btn{padding:12px 22px;border-radius:12px;background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border:1px solid rgba(255,255,255,0.06);backdrop-filter:blur(6px);color:white;font-weight:700;cursor:pointer;min-width:100px;text-align:center;box-shadow:0 6px 18px rgba(0,0,0,0.5)}
    .btn.neon{border-color:rgba(0,240,255,0.6);box-shadow:0 0 18px rgba(0,240,255,0.18);text-shadow:0 0 6px rgba(0,240,255,0.35)}
    .btn.danger{border-color:rgba(255,0,240,0.6);box-shadow:0 0 18px rgba(255,0,240,0.12)}
    .levels{position:absolute;bottom:18px;left:50%;transform:translateX(-50%);display:flex;gap:12px;z-index:21}
    .level-btn{padding:10px 14px;border-radius:10px;border:1px solid rgba(255,255,255,0.06);min-width:110px;text-align:center}
    #gameCanvas{width:100%;height:100%;display:block;background:linear-gradient(180deg,#071428 0%, #04101b 100%);}
    .hud{position:absolute;top:10px;left:10px;color:white;font-weight:700;z-index:25;text-shadow:0 0 6px rgba(0,0,0,0.6)}
    .rotate{position:absolute;inset:0;background:rgba(2,6,12,0.9);display:none;align-items:center;justify-content:center;color:#fff;z-index:40}
    @media (orientation:portrait){ .rotate{display:flex} .menu{display:none} }
    .controls-hint{position:absolute;right:10px;bottom:10px;padding:8px 10px;border-radius:8px;background:var(--glass);border:1px solid rgba(255,255,255,0.04);color:#cfe;z-index:25}
    @media (max-width:420px){ .title{font-size:36px} .btn{min-width:80px;padding:10px 14px} }
  </style>
</head>
<body>
  <div class="app">
    <div class="game-shell" id="shell">
      <div class="grid"></div>
      <canvas id="gameCanvas" width="1280" height="720"></canvas>
      <div class="menu" id="menu">
        <div class="title">NEON SKOKÁNCI</div>
        <div class="menu-row">
          <div class="btn neon" id="playBtn">Hrát</div>
          <div class="btn" id="settingsBtn">Nastavení</div>
          <div class="btn" id="editBtn">Edit</div>
          <div class="btn danger" id="quitBtn">Konec</div>
        </div>
      </div>
      <div class="levels" id="levelSelect" style="display:none;">
        <div class="level-btn btn neon" data-level="easy">Lehký</div>
        <div class="level-btn btn" data-level="medium">Střední</div>
        <div class="level-btn btn" data-level="hard">Těžký</div>
        <div class="level-btn btn danger" data-level="extreme">Extrémní</div>
      </div>
      <div class="hud" id="hud">Score: 0</div>
      <div class="controls-hint">Tap / Space — skok</div>
      <div class="rotate" id="rotateOverlay">
        <div style="text-align:center;max-width:80%"><h2>Otoč zařízení na šířku</h2><p>Hru spustěte v režimu naležato pro nejlepší zážitek (Redmi 10).</p></div>
      </div>
    </div>
  </div>
  <script>
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  const hud = document.getElementById('hud');
  const menu = document.getElementById('menu');
  const levelSelect = document.getElementById('levelSelect');
  const rotateOverlay = document.getElementById('rotateOverlay');
  const shell = document.getElementById('shell');
  function fitCanvas(){ const rect = shell.getBoundingClientRect(); canvas.width = 1280; canvas.height = 720; canvas.style.width = rect.width + 'px'; canvas.style.height = rect.height + 'px'; }
  fitCanvas();
  window.addEventListener('resize', fitCanvas);
  const COLORS = {bg:'#04101b', player:'#00f0ff', platform:'#7fffd4', spike:'#ff0044'};
  const player = {x:120, y:520, w:46, h:46, vy:0, onGround:false};
  const gravity = 1800;
  const jumpVel = -650;
  const levels = { easy:{speed:320,platforms:[{x:0,y:580,w:2000,h:140},{x:600,y:480,w:180,h:18},{x:950,y:420,w:220,h:18},{x:1400,y:520,w:160,h:18},{x:1800,y:460,w:220,h:18}],spikes:[{x:420,y:556,w:24,h:24},{x:560,y:556,w:24,h:24},{x:820,y:556,w:24,h:24},{x:1280,y:556,w:24,h:24}],length:2200}, medium:{speed:420,platforms:[{x:0,y:580,w:2000,h:140},{x:520,y:500,w:140,h:18},{x:760,y:420,w:120,h:18},{x:1020,y:360,w:160,h:18},{x:1500,y:520,w:140,h:18},{x:1700,y:440,w:120,h:18}],spikes:[{x:680,y:556,w:24,h:24},{x:900,y:556,w:24,h:24},{x:1180,y:556,w:24,h:24},{x:1380,y:496,w:24,h:24}],length:2200}, hard:{speed:560,platforms:[{x:0,y:580,w:2500,h:140},{x:420,y:520,w:100,h:18},{x:720,y:430,w:90,h:18},{x:950,y:370,w:120,h:18},{x:1230,y:480,w:140,h:18},{x:1600,y:420,w:120,h:18},{x:1900,y:360,w:120,h:18}],spikes:[{x:560,y:556,w:24,h:24},{x:840,y:536,w:24,h:24},{x:1100,y:556,w:24,h:24},{x:1500,y:496,w:24,h:24},{x:1760,y:396,w:24,h:24}],length:2600}, extreme:{speed:720,platforms:[{x:0,y:580,w:3000,h:140},{x:400,y:520,w:80,h:18},{x:640,y:460,w:80,h:18},{x:860,y:390,w:80,h:18},{x:1100,y:480,w:80,h:18},{x:1320,y:420,w:80,h:18},{x:1580,y:360,w:80,h:18},{x:1850,y:440,w:80,h:18}],spikes:[{x:520,y:556,w:24,h:24},{x:700,y:496,w:24,h:24},{x:880,y:436,w:24,h:24},{x:1040,y:396,w:24,h:24},{x:1260,y:356,w:24,h:24},{x:1480,y:496,w:24,h:24}],length:3000} };
  let state='menu'; let worldX=0; let curLevel=null; let score=0;
  let wantJump=false; function inputJump(){wantJump=true;}
  window.addEventListener('keydown',e=>{if(e.code==='Space'||e.code==='ArrowUp') inputJump(); if(e.key==='r'&&state==='died') resetToMenu();});
  window.addEventListener('touchstart',e=>{inputJump();},{passive:true});
  window.addEventListener('mousedown',e=>{inputJump();});
  document.getElementById('playBtn').addEventListener('click',()=>{menu.style.display='none'; levelSelect.style.display='flex'; state='levelSelect';});
  document.getElementById('settingsBtn').addEventListener('click',()=>{alert('Nastavení: zvuk a citlivost nejsou implementovány v této demo verzi.');});
  document.getElementById('editBtn').addEventListener('click',()=>{alert('Editor není implementován v tomto demo souboru.');});
  document.getElementById('quitBtn').addEventListener('click',()=>{if(confirm('Opravdu ukončit hru?')) window.close();});
  levelSelect.addEventListener('click',e=>{const el=e.target.closest('.level-btn'); if(!el) return; startLevel(el.dataset.level);});
  function startLevel(key){curLevel=JSON.parse(JSON.stringify(levels[key])); worldX=0; score=0; player.x=120; player.y=520; player.vy=0; player.onGround=false; state='playing'; levelSelect.style.display='none';}
  function resetToMenu(){state='menu'; menu.style.display='flex'; levelSelect.style.display='none';}
  function rectsOverlap(a,b){return a.x<b.x+b.w && a.x+a.w>b.x && a.y<b.y+b.h && a.y+a.h>b.y;}
  let last=performance.now(); function loop(now){const dt=Math.min((now-last)/1000,0.05); last=now; update(dt); render(); requestAnimationFrame(loop);} requestAnimationFrame(loop);
  function update(dt){rotateOverlay.style.display=(window.innerHeight>window.innerWidth)?'flex':'none'; if(state!=='playing') return; const speed=curLevel.speed; player.vy+=gravity*dt; player.y+=player.vy*dt; player.onGround=false; const playerRect={x:player.x,y:player.y,w:player.w,h:player.h}; for(const p of curLevel.platforms){const platRect={x:p.x-worldX,y:p.y,w:p.w,h:p.h}; if(rectsOverlap(playerRect,platRect)){if(player.vy>=0 && (player.y+player.h)-(platRect.y)<60){player.y=platRect.y-player.h; player.vy=0; player.onGround=true;}}} if(wantJump){if(player.onGround){player.vy=jumpVel; player.onGround=false;} wantJump=false;} worldX+=speed*dt; score=Math.floor(worldX/10); hud.textContent='Score: '+score; const playerWorldRect={x:player.x+worldX,y:player.y,w:player.w,h:player.h}; for(const s of curLevel.spikes){const spikeRect={x:s.x,y:s.y,w:s.w,h:s.h}; if(rectsOverlap(playerWorldRect,spikeRect)){state='died'; setTimeout(()=>{if(confirm('Prohrál jsi — restartovat úroveň?')) startLevel(getLevelKeyBySpeed(curLevel.speed)); else resetToMenu();},50);}} if(worldX>curLevel.length){state='victory'; setTimeout(()=>{alert('Gratulace! Úroveň dokončena.'); resetToMenu();},100);}}
  function getLevelKeyBySpeed(spd){for(const k in levels) if(levels[k].speed===spd) return k; return 'easy';}
  function render(){ctx.fillStyle=COLORS.bg; ctx.fillRect(0,0,canvas.width,canvas.height); ctx.fillStyle='#071428'; ctx.fillRect(0,0,canvas.width,canvas.height); ctx.save(); const scaleX=canvas.width/1280; const scaleY=canvas.height/720; ctx.scale(scaleX,scaleY); for(const p of (curLevel?curLevel.platforms:[])){const drawX=p.x-worldX; ctx.fillStyle='#0ff8e0'; ctx.fillRect(drawX,p.y,p.w,p.h); ctx.fillStyle='rgba(0,240,255,0.12)'; ctx.fillRect(drawX,p.y-6,p.w,6);} for(const s of (curLevel?curLevel.spikes:[])){const drawX=s.x-worldX; ctx.fillStyle='#ff0044'; ctx.beginPath(); ctx.moveTo(drawX,s.y+s.h); ctx.lineTo(drawX+s.w/2,s.y); ctx.lineTo(drawX+s.w,s.y+s.h); ctx.closePath(); ctx.fill();} ctx.fillStyle='#00f0ff'; ctx.fillRect(player.x,player.y,player.w,player.h); ctx.fillStyle='rgba(0,240,255,0.14)'; ctx.fillRect(player.x-6,player.y-6,player.w+12,player.h+12); ctx.fillStyle='rgba(255,255,255,0.02)'; ctx.fillRect(0,700,1280,20); ctx.restore();}
  window.__gd={startLevel,resetToMenu,levels};
  </script>
</body>
</html>
