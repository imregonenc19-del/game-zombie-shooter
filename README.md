<!DOCTYPE html>
<html lang="nl">
<head>
<meta charset="UTF-8">
<title>Zombie Shooter FPS ELITE</title>
<style>
body { margin:0; overflow:hidden; background:black; }
#ui { position:absolute; top:10px; left:10px; color:white; font-family:Arial; font-size:20px; }
#waveInfo { position:absolute; top:40px; left:10px; color:yellow; font-family:Arial; font-size:18px; }
#menu {
  position:absolute; top:0; left:0; width:100%; height:100%;
  display:flex; align-items:center; justify-content:center;
  background:black; color:white; font-size:30px; cursor:pointer;
}
#crosshair { position:absolute; top:50%; left:50%; width:10px; height:10px; margin-left:-5px; margin-top:-5px; border:2px solid white; border-radius:50%; }
#restart {
  display:none; position:absolute; top:50%; left:50%;
  transform:translate(-50%,-50%);
  background:black; color:white; padding:20px; font-size:25px; cursor:pointer;
}
#minimap { position:absolute; bottom:10px; right:10px; width:200px; height:200px; background:rgba(0,0,0,0.5); border:1px solid white; }
</style>
</head>
<body>
<div id="menu">KLIK OM TE STARTEN</div>
<div id="ui"></div>
<div id="waveInfo"></div>
<div id="crosshair"></div>
<div id="restart">GAME OVER<br>KLIK OM TE HERSTARTEN</div>
<canvas id="minimap"></canvas>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js"></script>
<script>
// --- Scene Setup ---
let scene = new THREE.Scene();
scene.fog = new THREE.FogExp2(0x000000,0.02);
let camera = new THREE.PerspectiveCamera(75,window.innerWidth/window.innerHeight,0.1,1000);
let renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(window.innerWidth,window.innerHeight);
document.body.appendChild(renderer.domElement);

// Lights
scene.add(new THREE.AmbientLight(0x404040));
let dirLight = new THREE.DirectionalLight(0xffffff,1);
dirLight.position.set(5,10,5); scene.add(dirLight);

// Floor
let floorTex = new THREE.TextureLoader().load('https://threejsfundamentals.org/threejs/resources/images/checker.png');
floorTex.wrapS = floorTex.wrapT = THREE.RepeatWrapping;
floorTex.repeat.set(20,20);
let floor = new THREE.Mesh(new THREE.PlaneGeometry(200,200), new THREE.MeshStandardMaterial({map:floorTex}));
floor.rotation.x=-Math.PI/2; scene.add(floor);

// Walls
let walls=[];
for(let i=0;i<30;i++){
  let w = new THREE.Mesh(new THREE.BoxGeometry(5,3,1), new THREE.MeshStandardMaterial({color:0x777777}));
  w.position.set((Math.random()-0.5)*100,1.5,(Math.random()-0.5)*100); scene.add(w); walls.push(w);
}

// Player
let player = new THREE.Object3D(); player.position.y=1.8; scene.add(player); player.add(camera);
let keys={}, bullets=[], zombies=[], powerups=[];
let hp=100, score=0, velocityY=0, canJump=true, fireRate=200, speedMultiplier=1;

// Gun
let gun = new THREE.Mesh(new THREE.BoxGeometry(0.3,0.2,1), new THREE.MeshStandardMaterial({color:0x111111}));
gun.position.set(0.5,-0.5,-1); camera.add(gun);

// Menu & Restart
let menu = document.getElementById("menu"), restartDiv=document.getElementById("restart");
menu.onclick = ()=>{ menu.style.display="none"; document.body.requestPointerLock(); startWave(); };
restartDiv.onclick = resetGame;

// Mini-map
let minimap=document.getElementById("minimap"), minimapCtx=minimap.getContext("2d");

// Mouse look
document.addEventListener("mousemove",e=>{
  if(document.pointerLockElement===document.body){
    player.rotation.y -= e.movementX*0.002;
    camera.rotation.x -= e.movementY*0.002;
    camera.rotation.x = Math.max(-Math.PI/2,Math.min(Math.PI/2,camera.rotation.x));
  }
});

// Keyboard
window.addEventListener("keydown",e=>{ keys[e.key.toLowerCase()]=true; if(e.code==="Space"&&canJump){ velocityY=0.2; canJump=false; }});
window.addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);

// Shooting
let lastShot=0;
window.addEventListener("click",()=>{
  if(document.pointerLockElement!==document.body) return;
  let now=performance.now(); if(now-lastShot<fireRate) return; lastShot=now;
  let bullet = new THREE.Mesh(new THREE.SphereGeometry(0.1), new THREE.MeshBasicMaterial({color:0xffff00}));
  bullet.position.copy(camera.position);
  let dir = new THREE.Vector3(); camera.getWorldDirection(dir); bullet.velocity = dir.clone().multiplyScalar(2);
  scene.add(bullet); bullets.push(bullet);
  gun.position.z+=0.1; setTimeout(()=>gun.position.z-=0.1,50);
});

// Wave system
let currentWave=1, zombiesRemaining=0;
function startWave(){
  zombiesRemaining = currentWave*5;
  document.getElementById("waveInfo").innerText=`Wave: ${currentWave}`;
  for(let i=0;i<zombiesRemaining;i++) spawnZombie();
}

// Spawn zombie
function spawnZombie(){
  let z = new THREE.Mesh(new THREE.CylinderGeometry(0.5,0.5,2,6), new THREE.MeshStandardMaterial({color:0x00aa00}));
  let safe=false; while(!safe){ z.position.set((Math.random()-0.5)*80,1,(Math.random()-0.5)*80); safe=z.position.distanceTo(player.position)>10; walls.forEach(w=>{ if(z.position.distanceTo(w.position)<3)safe=false; }); }
  z.hp=3; z.speed=0.04; z.scoreVal=1; scene.add(z); zombies.push(z);
}

// Power-ups
function spawnPowerup(){
  let p = new THREE.Mesh(new THREE.BoxGeometry(1,1,1), new THREE.MeshStandardMaterial({color:0x00ff00}));
  p.position.set((Math.random()-0.5)*80,0.5,(Math.random()-0.5)*80); scene.add(p); powerups.push(p);
}
setInterval(spawnPowerup,10000);

// Collision
function checkCollision(pos){ for(let w of walls){ if(Math.abs(pos.x-w.position.x)<3 && Math.abs(pos.z-w.position.z)<1.5) return true; } return false; }

// Reset
function resetGame(){
  bullets.forEach(b=>scene.remove(b)); bullets=[];
  zombies.forEach(z=>scene.remove(z)); zombies=[];
  powerups.forEach(p=>scene.remove(p)); powerups=[];
  player.position.set(0,1.8,0); hp=100; score=0; fireRate=200; speedMultiplier=1; restartDiv.style.display="none"; menu.style.display="none"; document.body.requestPointerLock();
  currentWave=1; startWave();
}

// Update
function update(){
  if(hp<=0) return;

  // Movement
  let speed = keys.shift?0.2:0.1; speed*=speedMultiplier;
  let move=new THREE.Vector3(); if(keys.w) move.z-=speed; if(keys.s) move.z+=speed; if(keys.a) move.x-=speed; if(keys.d) move.x+=speed;
  let angle=player.rotation.y;
  let dx=move.x*Math.cos(angle)-move.z*Math.sin(angle);
  let dz=move.x*Math.sin(angle)+move.z*Math.cos(angle);
  let newPos=player.position.clone(); newPos.x+=dx; newPos.z+=dz;
  if(!checkCollision(newPos)){ player.position.x=newPos.x; player.position.z=newPos.z; }

  // Gravity
  velocityY-=0.01; player.position.y+=velocityY; if(player.position.y<=1.8){ player.position.y=1.8; velocityY=0; canJump=true; }

  // Bullets
  bullets = bullets.filter(b=>{ if(b.velocity) b.position.add(b.velocity); if(b.position.length()>100){scene.remove(b); return false;} return true; });

  // Zombies
  zombies = zombies.filter(z=>{
    let dir=player.position.clone().sub(z.position).normalize(); z.position.add(dir.clone().multiplyScalar(z.speed));
    if(z.position.distanceTo(player.position)<1.5) hp-=0.3;
    for(let i=bullets.length-1;i>=0;i--){
      let b=bullets[i]; if(!b.velocity) continue;
      if(z.position.distanceTo(b.position)<1){ z.hp--; scene.remove(b); bullets.splice(i,1); if(z.hp<=0){ scene.remove(z); score+=z.scoreVal; zombiesRemaining--; return false; } }
    }
    return true;
  });

  // Power-ups
  powerups = powerups.filter(p=>{ if(player.position.distanceTo(p.position)<2){ hp=Math.min(hp+30,100); scene.remove(p); return false;} return true; });

  // Mini-map
  minimapCtx.clearRect(0,0,minimap.width,minimap.height);
  let px=(player.position.x+100)/200*minimap.width; let pz=(player.position.z+100)/200*minimap.height;
  minimapCtx.fillStyle="white"; minimapCtx.fillRect(px-3,pz-3,6,6);
  minimapCtx.fillStyle="red"; zombies.forEach(z=>{ let zx=(z.position.x+100)/200*minimap.width; let zz=(z.position.z+100)/200*minimap.height; minimapCtx.fillRect(zx-2,zz-2,4,4); });
}

// Animate
function animate(){
  requestAnimationFrame(animate);
  update();
  renderer.render(scene,camera);
  if(hp>0){ document.getElementById("ui").innerText=`HP:${Math.floor(hp)} Score:${score}`; document.getElementById("waveInfo").innerText=`Wave: ${currentWave}`; }
  else{ document.getElementById("ui").innerText="GAME OVER"; restartDiv.style.display="block"; }
}

// Wave interval
setInterval(()=>{ if(zombiesRemaining<=0&&hp>0){ currentWave++; startWave(); } },3000);

// Start game
animate();
</script>
</body>
</html>
