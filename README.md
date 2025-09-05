<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>GTA‑Lite • Top‑Down Mini City</title>
  <style>
    html, body { height: 100%; margin: 0; background:#111; color:#fff; font-family: system-ui,Segoe UI,Roboto,Helvetica,Arial,sans-serif; }
    #ui { position: fixed; inset: 0; pointer-events: none; }
    #hud { position: absolute; top: 12px; left: 12px; display:flex; gap:10px; align-items:center; }
    .pill { background: rgba(0,0,0,.55); border:1px solid rgba(255,255,255,.12); backdrop-filter: blur(6px); padding:6px 10px; border-radius: 999px; font-weight: 700; letter-spacing:.3px; }
    #msg { position:absolute; bottom: 16px; left: 50%; transform: translateX(-50%); padding:10px 14px; background: rgba(0,0,0,.55); border:1px solid rgba(255,255,255,.1); border-radius: 10px; pointer-events:none; max-width: min(92vw,720px); text-align:center; }
    #help { position:absolute; top: 12px; right: 12px; background: rgba(0,0,0,.55); border:1px solid rgba(255,255,255,.12); border-radius:12px; padding:10px; line-height:1.4; font-size:12px; max-width: 320px; }

    /* Virtual controls */
    .stick { position: fixed; width: 120px; height: 120px; border-radius: 50%; background: rgba(255,255,255,.06); border:1px solid rgba(255,255,255,.1); bottom: 18px; touch-action: none; pointer-events:auto; }
    .knob { position: absolute; width: 60px; height: 60px; border-radius: 50%; background: rgba(255,255,255,.35); border:1px solid rgba(255,255,255,.25); left: 30px; top: 30px; }
    #stickL { left: 18px; }
    #stickR { right: 18px; }

    .btn { position: fixed; bottom: 24px; right: 160px; width: 56px; height: 56px; border-radius: 50%; background: rgba(255,255,255,.15); border:1px solid rgba(255,255,255,.25); display:grid; place-items:center; font-weight:800; pointer-events:auto; user-select:none; }
    .btn2 { right: 228px; }

    @media (hover:hover) and (pointer:fine) { /* hide touch controls on desktop */
      #stickL, #stickR, .btn { display:none; }
    }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <div id="ui">
    <div id="hud">
      <div class="pill" id="mode">ON FOOT</div>
      <div class="pill" id="speed">0 km/h</div>
      <div class="pill" id="wanted">★☆☆☆☆</div>
      <div class="pill" id="health">♥ 100</div>
    </div>
    <div id="help">
      <b>Controls (Desktop)</b><br>
      WASD / Arrow Keys — Move / Drive<br>
      Shift — Sprint / Handbrake<br>
      E — Enter/Exit car (near door)<br>
      Space — Brake/Reverse (vehicle)<br>
      M — Mini‑mission (go to marker)<br>
      P — Spawn police pursuit (for fun)<br>
      R — Reset to spawn
    </div>
    <div id="msg">Welcome to <b>GTA‑Lite</b> — a tiny, asset‑free city. Reach the yellow marker when a mission is active. Cause trouble to raise your wanted level… and the cops will notice. 😉</div>

    <!-- Touch controls -->
    <div id="stickL" class="stick"><div class="knob"></div></div>
    <div id="stickR" class="stick"><div class="knob"></div></div>
    <div id="btnE" class="btn" title="Enter/Exit">E</div>
    <div id="btnHB" class="btn btn2" title="Handbrake">HB</div>
  </div>

<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const hud = {
    mode: document.getElementById('mode'),
    speed: document.getElementById('speed'),
    wanted: document.getElementById('wanted'),
    health: document.getElementById('health'),
    msg: document.getElementById('msg')
  };

  const DPR = Math.min(2, window.devicePixelRatio || 1);
  function resize() {
    canvas.width = Math.floor(innerWidth * DPR);
    canvas.height = Math.floor(innerHeight * DPR);
    canvas.style.width = innerWidth + 'px';
    canvas.style.height = innerHeight + 'px';
    ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  }
  addEventListener('resize', resize); resize();

  // World config
  const TILE = 48; // world tile size in px
  const CITY_W = 120; // tiles
  const CITY_H = 120;
  const world = { roads: new Set(), buildings: new Set(), sidewalks: new Set() };

  function tileKey(x,y){ return x+','+y; }
  function inCity(x,y){ return x>=0 && y>=0 && x<CITY_W && y<CITY_H; }

  // Generate a grid city: avenues every 6 tiles, streets every 6 tiles
  (function genCity(){
    for(let y=0;y<CITY_H;y++){
      for(let x=0;x<CITY_W;x++){
        const avenue = x%6===0;
        const street = y%6===0;
        if(avenue || street){
          world.roads.add(tileKey(x,y));
          // mark sidewalks adjacent to roads
          [[1,0],[-1,0],[0,1],[0,-1]].forEach(([dx,dy])=>{
            const sx=x+dx, sy=y+dy; if(inCity(sx,sy)) world.sidewalks.add(tileKey(sx,sy));
          });
        } else {
          world.buildings.add(tileKey(x,y));
        }
      }
    }
  })();

  // Utilities
  const rand = (a,b)=>a+Math.random()*(b-a);
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
  const dist=(ax,ay,bx,by)=>Math.hypot(ax-bx,ay-by);

  // Camera
  const camera = { x: CITY_W*TILE/2, y:CITY_H*TILE/2, zoom:1 };

  // Entities
  let idGen=1;
  const ALL=[]; const VEH=[]; const PED=[]; const POLICE=[];
  function add(ent){ ent.id=idGen++; ALL.push(ent); return ent; }

  // Player
  const player = add({ type:'ped', x: (CITY_W/2)*TILE+24, y:(CITY_H/2)*TILE+24, r:12, speed:0, angle:0, onFoot:true, health:100, wanted:0, in:null });
  PED.push(player);

  // Spawns
  function randomRoadXY(){
    let x=0,y=0; while(true){ x=(Math.random()*CITY_W)|0; y=(Math.random()*CITY_H)|0; if(world.roads.has(tileKey(x,y))) break; }
    return { x:x*TILE+TILE/2, y:y*TILE+TILE/2 };
  }

  // Vehicles
  function spawnCar(kind='civil', xy=randomRoadXY()){
    const car = add({ type:'car', kind, x:xy.x, y:xy.y, w:44, h:24, angle: rand(0,Math.PI*2), speed:0, max: kind==='police'? 6:5, accel:.15, turn:.05, ai:true, target:null, sirenPhase:0 });
    VEH.push(car); if(kind==='police') POLICE.push(car);
    return car;
  }

  // Pedestrians
  function spawnPed(xy){
    const p = add({ type:'ped', x:xy.x, y:xy.y, r:10, speed:0, angle:rand(0,Math.PI*2), onFoot:true, ai:true, cooldown:0, health:100 });
    PED.push(p); return p;
  }

  // Populate city
  for(let i=0;i<22;i++) spawnCar('civil');
  for(let i=0;i<40;i++) spawnPed(randomRoadXY());

  // Mission
  let mission=null; function newMission(){ mission = { active:true, target: randomRoadXY() }; say('Mini‑mission started! Reach the yellow marker (M to toggle).'); }

  // Input
  const keys = new Set();
  addEventListener('keydown', e=>{ keys.add(e.key.toLowerCase()); if(['Space','ArrowUp','ArrowDown','ArrowLeft','ArrowRight'].includes(e.code)) e.preventDefault();
    if(e.key.toLowerCase()==='e') tryEnterExit();
    if(e.key.toLowerCase()==='m'){ mission = mission && mission.active? null : (newMission(), mission); }
    if(e.key.toLowerCase()==='p'){ spawnPoliceChase(); }
    if(e.key.toLowerCase()==='r'){ resetToSpawn(); }
  });
  addEventListener('keyup', e=> keys.delete(e.key.toLowerCase()));

  function resetToSpawn(){ const c = CITY_W*TILE/2; player.x=c; player.y=c; player.in=null; player.onFoot=true; say('Back to spawn.'); }

  // Touch Controls (dual virtual sticks + buttons)
  function makeStick(el){
    const knob = el.querySelector('.knob');
    const state = { dx:0, dy:0, active:false };
    let startX=0, startY=0;
    const maxR = 45;
    function setKnob(x,y){ knob.style.left=(30+x-30)+'px'; knob.style.top=(30+y-30)+'px'; }
    const on = (type,fn)=> el.addEventListener(type,fn,{passive:false});
    on('touchstart',ev=>{ state.active=true; const t=ev.changedTouches[0]; const r=el.getBoundingClientRect(); startX=t.clientX; startY=t.clientY; ev.preventDefault(); });
    on('touchmove',ev=>{ if(!state.active) return; const t=ev.changedTouches[0]; const dx=clamp(t.clientX-startX,-maxR,maxR); const dy=clamp(t.clientY-startY,-maxR,maxR); state.dx=dx/maxR; state.dy=dy/maxR; setKnob(30+dx*0.66,30+dy*0.66); ev.preventDefault(); });
    on('touchend',()=>{ state.active=false; state.dx=state.dy=0; setKnob(30,30); });
    return state;
  }
  const stickL = makeStick(document.getElementById('stickL'));
  const stickR = makeStick(document.getElementById('stickR'));
  document.getElementById('btnE').addEventListener('touchstart', e=>{ e.preventDefault(); tryEnterExit(); });
  document.getElementById('btnHB').addEventListener('touchstart', e=>{ e.preventDefault(); handbrakePulse=8; });
  let handbrakePulse=0;

  function keyHeld(k){ return keys.has(k); }

  function say(text, ms=3000){ hud.msg.textContent = text; clearTimeout(say._t); say._t=setTimeout(()=>{hud.msg.textContent='';}, ms); }

  // Collision helpers
  function isBuildingAt(px,py){
    const tx = Math.floor(px/TILE), ty = Math.floor(py/TILE);
    return world.buildings.has(tileKey(tx,ty));
  }
  function isRoadAt(px,py){ const tx=(px/TILE)|0, ty=(py/TILE)|0; return world.roads.has(tileKey(tx,ty)); }
  function isSidewalkAt(px,py){ const tx=(px/TILE)|0, ty=(py/TILE)|0; return world.sidewalks.has(tileKey(tx,ty)); }

  // Enter / Exit vehicle
  function tryEnterExit(){
    if(player.in){ // exit
      player.onFoot=true; player.in.driver=null; player.in=null; say('Exited vehicle'); return;
    }
    // find nearest car door
    let best=null, bd=9999; for(const c of VEH){ const d = dist(player.x,player.y,c.x,c.y); if(d<36 && d<bd && !c.driver){ best=c; bd=d; }}
    if(best){ player.in=best; best.driver=player; player.onFoot=false; say('Entered '+(best.kind==='police'?'police car':'vehicle')); }
    else say('No empty car nearby');
  }

  // Wanted system
  function addWanted(x=1){ player.wanted = clamp(player.wanted + x, 0, 5); }
  function decayWanted(dt){ player.wanted = Math.max(0, player.wanted - dt*0.02); }
  function spawnPoliceChase(){ const xy = randomRoadXY(); const car = spawnCar('police', xy); car.ai=true; addWanted(3); say('Police alerted!'); }

  // AI
  function aiDrive(car, dt){
    // simple lane following: steer towards nearest road center along current angle
    // pick a forward look point along grid
    const speedBias = (car.kind==='police' && player.wanted>0)? 1.15:1;
    car.max = (car.kind==='police'? 6.5:5) * speedBias;

    // chase if police and wanted
    let target = null;
    if(car.kind==='police' && player.wanted>0){
      if(player.in && player.in===car) return; // if player is driving this police car
      target = player.in ? player.in : player; // chase person or vehicle
    }

    if(target){
      const dx = target.x - car.x; const dy = target.y - car.y; const ta = Math.atan2(dy, dx);
      let da = ((ta - car.angle + Math.PI*3)%(Math.PI*2))-Math.PI; // shortest
      car.angle += clamp(da, -car.turn, car.turn);
      if(isRoadAt(car.x,car.y)) car.speed = clamp(car.speed + car.accel, -car.max, car.max);
    } else {
      // wander along roads by snapping to nearest road direction, turn at intersections
      const gridX = Math.round(car.x / TILE), gridY = Math.round(car.y / TILE);
      const onIntersection = world.roads.has(tileKey(gridX,gridY)) && world.roads.has(tileKey(gridX+1,gridY)) && world.roads.has(tileKey(gridX,gridY+1)) && world.roads.has(tileKey(gridX-1,gridY)) && world.roads.has(tileKey(gridX,gridY-1));
      if(onIntersection && Math.random()<0.02){ car.angle = [0,Math.PI/2,Math.PI,Math.PI*1.5][(Math.random()*4)|0]; }
      // steer softly to align to 0/90°
      const desired = Math.round(car.angle/(Math.PI/2))*(Math.PI/2);
      let da = ((desired - car.angle + Math.PI*3)%(Math.PI*2))-Math.PI;
      car.angle += clamp(da, -car.turn*0.6, car.turn*0.6);
      if(isRoadAt(car.x,car.y)) car.speed = clamp(car.speed + car.accel*0.6, -car.max*0.6, car.max*0.6);
    }

    // friction
    car.speed *= 0.992;

    // move and basic wall hit
    const nx = car.x + Math.cos(car.angle)*car.speed;
    const ny = car.y + Math.sin(car.angle)*car.speed;
    if(!isBuildingAt(nx,ny)) { car.x=nx; car.y=ny; }

    // siren
    if(car.kind==='police' && player.wanted>0){ car.sirenPhase += dt*6; }
  }

  function aiPed(p,dt){
    if(p===player) return;
    p.cooldown -= dt;
    if(p.cooldown<=0){
      // choose new direction occasionally
      p.angle += rand(-1,1);
      p.cooldown = rand(0.6, 2.0);
    }
    const speed = 1.2;
    const nx = p.x + Math.cos(p.angle)*speed;
    const ny = p.y + Math.sin(p.angle)*speed;
    if(isSidewalkAt(nx,ny)) { p.x=nx; p.y=ny; } else { p.angle += Math.PI/2; }
  }

  // Update
  let last=performance.now();
  function loop(t){ const dt = Math.min(0.05, (t-last)/1000); last=t; update(dt); draw(); requestAnimationFrame(loop); }
  requestAnimationFrame(loop);

  function update(dt){
    // Player input
    if(player.onFoot){
      const dx = (keyHeld('d')||keyHeld('arrowright')?1:0) - (keyHeld('a')||keyHeld('arrowleft')?1:0) + (stickL.active? stickL.dx:0);
      const dy = (keyHeld('s')||keyHeld('arrowdown')?1:0) - (keyHeld('w')||keyHeld('arrowup')?1:0) + (stickL.active? stickL.dy:0);
      const mag = Math.hypot(dx,dy);
      const run = keyHeld('shift')? 1.6 : 1;
      const spd = clamp((mag? 2.4:0) * run, 0, 3.2);
      if(mag){ player.angle = Math.atan2(dy, dx); }
      const nx = player.x + Math.cos(player.angle)*spd;
      const ny = player.y + Math.sin(player.angle)*spd;
      if(!isBuildingAt(nx,ny)) { player.x=nx; player.y=ny; }
    } else if(player.in){
      const car = player.in;
      // drive
      const steer = ((keyHeld('a')||keyHeld('arrowleft'))? -1:0) + ((keyHeld('d')||keyHeld('arrowright'))? 1:0) + (stickL.active? stickL.dx:0);
      const throttle = ((keyHeld('w')||keyHeld('arrowup'))? 1:0) + (stickR.active? Math.max(0,-stickR.dy):0);
      const brake = ((keyHeld('s')||keyHeld('arrowdown')||keyHeld(' '))? 1:0) + (stickR.active? Math.max(0,stickR.dy):0);
      const handbrake = keyHeld('shift') || handbrakePulse>0; if(handbrakePulse>0) handbrakePulse--;

      // turn scales with speed
      car.angle += steer * car.turn * (0.6 + Math.min(1, Math.abs(car.speed)/car.max));

      // accelerate / brake
      if(throttle) car.speed = clamp(car.speed + car.accel*2*throttle, -car.max, car.max);
      if(brake) car.speed = clamp(car.speed - car.accel*2.2*brake, -car.max*0.5, car.max);
      if(handbrake) car.speed *= 0.92;

      // move
      const nx = car.x + Math.cos(car.angle)*car.speed;
      const ny = car.y + Math.sin(car.angle)*car.speed;
      if(!isBuildingAt(nx,ny)) { car.x=nx; car.y=ny; } else { car.speed*=-0.4; addWanted(0.2); }
      player.x = car.x; player.y = car.y; player.angle = car.angle;
    }

    // Update AI
    for(const c of VEH){ aiDrive(c, dt); }
    for(const p of PED){ aiPed(p, dt); }

    // Simple collisions: car vs ped (crime!)
    for(const c of VEH){
      for(const p of PED){ if(p===player && player.onFoot===false) continue; // don't hit yourself
        const d = dist(c.x,c.y,p.x,p.y); if(d < 16){
          if(p===player){ player.health -= Math.min(25, Math.abs(c.speed*6)); if(player.health<=0){ say('Wasted. Respawning…'); player.health=100; resetToSpawn(); addWanted(-5); } }
          else { // NPC ped
            p.x += (p.x-c.x)*0.6; p.y += (p.y-c.y)*0.6; if(Math.random()<0.3) addWanted(0.5);
          }
        }
      }
    }

    // Wanted decay
    decayWanted(dt);

    // Mission check
    if(mission && mission.active){ if(dist(player.x,player.y, mission.target.x, mission.target.y) < 28){ say('Mission complete! +$500 (imaginary)'); mission=null; } }

    // Camera follow
    const follow = player.in || player; camera.x += (follow.x - camera.x)*0.15; camera.y += (follow.y - camera.y)*0.15;

    // HUD
    hud.mode.textContent = player.onFoot? 'ON FOOT' : (player.in && player.in.kind==='police'? 'DRIVING • POLICE' : 'DRIVING');
    const v = (player.in? Math.abs(player.in.speed*18) : Math.hypot(0,0));
    hud.speed.textContent = (v|0) + ' km/h';
    const stars = Math.round(player.wanted);
    hud.wanted.textContent = '★★★★★'.slice(0,stars)+'☆☆☆☆☆'.slice(stars,5);
    hud.health.textContent = '♥ '+(player.health|0);
  }

  function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // compute view bounds
    const vw = canvas.width/DPR, vh = canvas.height/DPR;
    const left = camera.x - vw/2, top = camera.y - vh/2;

    // grid background
    ctx.save();
    ctx.translate(-left, -top);

    // Draw tiles
    const startX = Math.max(0, Math.floor(left/TILE)-1);
    const startY = Math.max(0, Math.floor(top/TILE)-1);
    const endX = Math.min(CITY_W, Math.ceil((left+vw)/TILE)+1);
    const endY = Math.min(CITY_H, Math.ceil((top+vh)/TILE)+1);

    for(let y=startY;y<endY;y++){
      for(let x=startX;x<endX;x++){
        const k=tileKey(x,y);
        const px=x*TILE, py=y*TILE;
        if(world.roads.has(k)){
          ctx.fillStyle = '#2d2d2d';
          ctx.fillRect(px,py,TILE,TILE);
          // lane dots
          ctx.strokeStyle = 'rgba(255,255,255,.2)';
          ctx.lineWidth = 2; ctx.setLineDash([8,8]);
          ctx.beginPath(); ctx.moveTo(px, py+TILE/2); ctx.lineTo(px+TILE, py+TILE/2); ctx.stroke();
          ctx.beginPath(); ctx.moveTo(px+TILE/2, py); ctx.lineTo(px+TILE/2, py+TILE); ctx.stroke();
          ctx.setLineDash([]);
        } else if(world.sidewalks.has(k)){
          ctx.fillStyle = '#3b3b3b';
          ctx.fillRect(px,py,TILE,TILE);
        } else {
          ctx.fillStyle = '#1a1a1a';
          ctx.fillRect(px,py,TILE,TILE);
          // simple rooftops
          ctx.fillStyle = 'rgba(255,255,255,.05)';
          ctx.fillRect(px+6,py+6,TILE-12,TILE-12);
        }
      }
    }

    // Mission marker
    if(mission && mission.active){
      ctx.fillStyle = 'rgba(255,255,0,.5)';
      ctx.beginPath(); ctx.arc(mission.target.x, mission.target.y, 18, 0, Math.PI*2); ctx.fill();
      ctx.strokeStyle = 'rgba(255,255,0,.9)'; ctx.lineWidth=2; ctx.stroke();
    }

    // Draw vehicles
    for(const c of VEH){
      ctx.save(); ctx.translate(c.x, c.y); ctx.rotate(c.angle);
      ctx.fillStyle = c.kind==='police'? '#204070' : '#752222';
      ctx.fillRect(-c.w/2,-c.h/2,c.w,c.h);
      // windows
      ctx.fillStyle = '#bcd'; ctx.fillRect(-c.w/2+4, -c.h/2+3, c.w-8, 7);
      ctx.fillRect(-c.w/2+4,  c.h/2-10, c.w-8, 7);
      // lights (police)
      if(c.kind==='police' && player.wanted>0){
        const phase = Math.sin(c.sirenPhase);
        ctx.fillStyle = phase>0? 'rgba(255,0,0,.8)':'rgba(0,120,255,.8)';
        ctx.fillRect(-c.w/2, -c.h/2-3, c.w/2, 3);
        ctx.fillStyle = phase>0? 'rgba(0,120,255,.8)':'rgba(255,0,0,.8)';
        ctx.fillRect(0, -c.h/2-3, c.w/2, 3);
      }
      ctx.restore();
    }

    // Draw pedestrians
    for(const p of PED){
      ctx.save(); ctx.translate(p.x, p.y);
      ctx.fillStyle = (p===player)? '#fff' : '#8fd18f';
      ctx.beginPath(); ctx.arc(0,0,p.r,0,Math.PI*2); ctx.fill();
      ctx.restore();
    }

    ctx.restore();
  }

  // Spawn a couple of police cars parked at spawn block
  for(let i=0;i<2;i++) spawnCar('police', {x: player.x+rand(-80,80), y: player.y+rand(-80,80)});

  say('Press M to start a mini‑mission. E to enter a car.');
})();
</script>
</body>
</html>
