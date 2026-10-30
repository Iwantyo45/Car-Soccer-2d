# Car-Soccer-2d
1v1 vs a bot

<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Turbo Goal</title>
<style>
body {
  margin: 0;
  background: #101827;
  color: white;
  font-family: system-ui, sans-serif;
}
#game {
  max-width: 900px;
  margin: auto;
  background: #101827;
  border-radius: 14px;
  overflow: hidden;
}
header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  background: #1b2940;
}
button {
  cursor: pointer;
  border: 0;
  border-radius: 8px;
  padding: 10px;
  background: #334155;
  color: white;
}
canvas {
  width: 100%;
  display: block;
  background: #16313b;
}
#overlay {
  position: absolute;
  inset: 0;
  background: #0b1225ed;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 14px;
}
#overlayContent {
  width: 100%;
  max-width: 540px;
  text-align: center;
}
#modes {
  display: grid;
  grid-template-columns: repeat(2,1fr);
  gap: 9px;
}
.mode {
  border: 1px solid #40516d;
  border-radius: 10px;
  padding: 12px;
  text-align: left;
  background: #1b2940;
  color: white;
  min-height: 65px;
}
#screen {
  position: relative;
}
footer {
  padding: 10px;
  display: flex;
  justify-content: space-between;
  gap: 8px;
  flex-wrap: wrap;
  color: #cbd5e1;
  font-size: 13px;
}
</style>
</head>
<body>

<div id="game">
<header>
<b>⚡ TURBO GOAL</b>
<span id="modeLabel">HAUPTMENÜ</span>
<button id="menuButton">MENÜ</button>
</header>

<div id="screen">
<canvas id="canvas" width="900" height="510"></canvas>

<div id="overlay">
<div id="overlayContent">
<h1>TURBO <span style="color:#fbbf24">GOAL</span></h1>
<p id="description">Wähle deinen Spielmodus</p>
<div id="modes"></div>
<p style="font-size:12px;color:#cbd5e1">
WASD fahren · SHIFT Boost · SPACE Power-up
</p>
</div>
</div>
</div>

<footer>
<span id="status">Modus auswählen</span>
<span id="boost">P1 Boost: 100%</span>
</footer>
</div>

<script>
const canvas = document.getElementById("canvas");
const ctx = canvas.getContext("2d");

const W = 900;
const H = 510;

const overlay = document.getElementById("overlay");
const modes = document.getElementById("modes");
const status = document.getElementById("status");
const label = document.getElementById("modeLabel");

const choices = [
 ["bot","🤖 1 VS BOT","Gegen Computer"],
 ["local","👥 2 SPIELER","Zusammen an einer Tastatur"],
 ["rumble","🎲 RUMBLE","Zufällige Power-ups"],
 ["big","🪐 BIG BALL","Riesiger Ball"],
 ["low","🌙 LOW GRAVITY","Leichter fliegen"],
 ["speed","⚡ SUPER SPEED","Schnelle Autos"],
 ["demo","💥 DEMOLITION","Gegner rammen"]
];

let keys = {};
let mode = "bot";
let playing = false;
let last = 0;
let score = [0,0];
let time = 90;
let freeze = 0;
let cd = [0,0];
let power = "";
let particles = [];

let cars = [];
let ball;

const clamp = (x,a,b) => Math.max(a,Math.min(b,x));

function car(x,y,color,angle,bot) {
 return {
  x,y,vx:0,vy:0,a:angle,color,bot,
  boost:100,stun:0
 };
}

function reset() {
 cars = [
  car(155,H/2,"#39c8ff",0,false),
  car(W-155,H/2,"#ff6658",Math.PI,mode !== "local")
 ];

 ball = {
  x:W/2,
  y:H/2,
  vx:0,
  vy:0,
  r:mode === "big" ? 28 : 12
 };

 freeze = 1;
 cd = [0,0];
 power = "";
}

function menu() {
 playing = false;
 overlay.style.display = "flex";
 label.textContent = "HAUPTMENÜ";
 status.textContent = "Modus auswählen";
 document.getElementById("description").textContent =
 "Wähle deinen Spielmodus";
}

function start(m) {
 mode = m;
 score = [0,0];
 time = 90;
 particles = [];
 reset();

 playing = true;
 overlay.style.display = "none";

 label.textContent =
 choices.find(a => a[0] === m)[1];

 status.textContent =
 mode === "local"
 ? "P1: WASD · P2: PFEILTASTEN"
 : "WASD · SHIFT Boost · SPACE Power";

 last = performance.now();
}

choices.forEach(a => {
 let b = document.createElement("button");
 b.className = "mode";
 b.innerHTML =
 "<b style='color:#7dd3fc'>" + a[1] +
 "</b><div style='font-size:12px;margin-top:4px'>" +
 a[2] + "</div>";

 b.onclick = () => start(a[0]);
 modes.appendChild(b);
});

document.getElementById("menuButton").onclick = menu;

window.addEventListener("keydown",e => {
 let k = e.key.toLowerCase();
 keys[k] = true;

 if([" ","arrowup","arrowdown","arrowleft","arrowright"].includes(k))
 e.preventDefault();

 if(k === "escape") menu();

 if(k === "r" && playing) start(mode);

 if(k === " " && !e.repeat && playing)
 usePower(0);

 if(k === "enter" && !e.repeat && playing && mode === "local")
 usePower(1);
});

window.addEventListener("keyup",e => {
 keys[e.key.toLowerCase()] = false;
});

function usePower(i) {
 if(cd[i] > 0) return;

 let p = cars[i];

 if(mode === "rumble") {
  let a = ["freeze","grapple","pulse","turbo"]
   [Math.floor(Math.random()*4)];

  power = a;
  cd[i] = 7;

  if(a === "freeze") {
   ball.vx *= .15;
   ball.vy *= .15;
   freeze = .6;
  }

  if(a === "pulse") {
   cars[1-i].stun = 1.2;
   ball.vx += ball.x < W/2 ? 180 : -180;
  }

  if(a === "grapple") {
   let dx = ball.x-p.x;
   let dy = ball.y-p.y;
   let d = Math.hypot(dx,dy)||1;

   p.vx += dx/d*300;
   p.vy += dy/d*300;
  }

  if(a === "turbo") {
   p.boost = 100;
   p.vx += Math.cos(p.a)*220;
   p.vy += Math.sin(p.a)*220;
  }

  status.textContent = "POWER-UP: " + a.toUpperCase();

 } else if(mode === "demo") {
  let q = cars[1-i];
  let dx = q.x-p.x;
  let dy = q.y-p.y;
  let d = Math.hypot(dx,dy)||1;

  q.vx += dx/d*280;
  q.vy += dy/d*280;
  q.stun = .7;
  cd[i] = 2;

 } else if(mode === "low") {
  p.vy -= 250;
  cd[i] = 1;

 } else {
  p.boost = 100;
  p.vx += Math.cos(p.a)*200;
  p.vy += Math.sin(p.a)*200;
  cd[i] = 2;
 }
}

function burst(x,y,col) {
 for(let i=0;i<25;i++) {
  let a = Math.random()*6.28;
  let s = 50+Math.random()*250;

  particles.push({
   x,y,
   vx:Math.cos(a)*s,
   vy:Math.sin(a)*s,
   t:.7,
   col
  });
 }
}

function goal(i) {
 score[i]++;

 burst(i===0 ? W-30 : 30,H/2,
       i===0 ? "#ff826d" : "#5ddcff");

 reset();
 freeze = 1.4;

 status.textContent = i===0 ? "TOR BLAU!" : "TOR ROT!";
}

function physicsCar(p,dt,i) {

 if(p.stun > 0) {
  p.stun -= dt;
  p.vx *= .95;
  p.vy *= .95;
  return;
 }

 let local = mode === "local";

 if(p.bot && !local) {

  let tx = ball.x + (ball.x>0 ? -1.2 : 1.2);
  let tz = ball.y;

  let aim = Math.atan2(tz-p.y,tx-p.x);
  let dif = Math.atan2(
   Math.sin(aim-p.a),
   Math.cos(aim-p.a)
  );

  p.a += clamp(dif,-2.8*dt,2.8*dt);

  p.vx += Math.cos(p.a)*8*dt;
  p.vy += Math.sin(p.a)*8*dt;

 } else {

  let l = i===0 ? "a" : "arrowleft";
  let r = i===0 ? "d" : "arrowright";
  let f = i===0 ? "w" : "arrowup";
  let b = i===0 ? "s" : "arrowdown";

  let turn = (keys[r]?1:0)-(keys[l]?1:0);
  let th = (keys[f]?1:0)-(keys[b]?1:0);

  p.a += turn*2.7*dt*(th<0?-1:1);

  let boost = i===0
   ? (keys.shift || keys.shiftleft)
   : keys.shift;

  let power = mode==="speed" ? 18 : 11;

  p.vx += Math.cos(p.a)*th*power*dt;
  p.vy += Math.sin(p.a)*th*power*dt;

  if(boost && th!==0 && p.boost>0) {
   p.vx += Math.cos(p.a)*20*dt*th;
   p.vy += Math.sin(p.a)*20*dt*th;
   p.boost -= 32*dt;
  } else {
   p.boost = Math.min(100,p.boost+10*dt);
  }

  if(i===0 && keys[" "] && p.y<.1)
   p.vy = mode==="low" ? 10 : 7.5;
 }

 p.vy -= (mode==="low" ? 3 : 15)*dt;
 p.y += p.vy*dt;

 if(p.y<0) {
  p.y=0;
  p.vy=0;
 }

 p.vx *= Math.pow(.984,dt*60);
 p.vy *= Math.pow(.984,dt*60);

 let s = Math.hypot(p.vx,p.vy);
 let max = mode==="speed" ? 16 : 11;

 if(s>max) {
  p.vx *= max/s;
  p.vy *= max/s;
 }

 p.x = clamp(p.x+p.vx*dt,-17,17);
 p.z = clamp(p.z+p.vy*dt,-8,8);
}

function update(dt) {
 if(!playing) return;

 time -= dt;

 if(time<=0) {
  time=0;
  playing=false;
  overlay.style.display="flex";

  document.getElementById("description").textContent =
   "Endstand "+score.join(" : ")+" · Rematch?";

  return;
 }

 physicsCar(cars[0],dt,0);
 physicsCar(cars[1],dt,1);

 if(keys[" "] && mode!=="low" &&
    mode!=="local" && cd[0]<=0) {
  usePower(0);
 }

 cd = cd.map(v=>Math.max(0,v-dt));

 if(freeze>0) {
  freeze-=dt;
  draw();
  return;
 }

 let dx = cars[1].x-cars[0].x;
 let dy = cars[1].y-cars[0].y;
 let d = Math.hypot(dx,dy)||1;

 if(d<34) {
  cars[0].vx -= dx/d*70;
  cars[0].vy -= dy/d*70;
  cars[1].vx += dx/d*70;
  cars[1].vy += dy/d*70;

  if(mode==="demo" &&
     Math.hypot(cars[0].vx,cars[0].vy)>180)
   cars[1].stun=.9;
 }

 ball.vx *= Math.pow(.996,dt*60);
 ball.vy *= Math.pow(.996,dt*60);

 ball.x += ball.vx*dt;
 ball.y += ball.vy*dt;

 let r = ball.r;

 if(ball.y>H/2-76 && ball.y<H/2+76) {
  if(ball.x-r<20 && ball.vx<0) {
   goal(1);
   draw();
   return;
  }

  if(ball.x+r>W-20 && ball.vx>0) {
   goal(0);
   draw();
   return;
  }
 }

 if(ball.y<r+18) {
  ball.y=r+18;
  ball.vy=Math.abs(ball.vy)*.83;
 }

 if(ball.y>H-r-18) {
  ball.y=H-r-18;
  ball.vy=-Math.abs(ball.vy)*.83;
 }

 if(ball.x<r+18) {
  ball.x=r+18;
  ball.vx=Math.abs(ball.vx)*.83;
 }

 if(ball.x>W-r-18) {
  ball.x=W-r-18;
  ball.vx=-Math.abs(ball.vx)*.83;
 }

 cars.forEach(p=>{
  dx=ball.x-p.x;
  dy=ball.y-p.y;

  d=Math.hypot(dx,dy)||1;

  if(d<r+17) {
   let nx=dx/d;
   let ny=dy/d;

   ball.x=p.x+nx*(r+18);
   ball.y=p.y+ny*(r+18);

   ball.vx += nx*(150+Math.hypot(p.vx,p.vy)*.8)+p.vx*.4;
   ball.vy += ny*(150+Math.hypot(p.vx,p.vy)*.8)+p.vy*.4;

   let sp=Math.hypot(ball.vx,ball.vy);

   if(sp>600) {
    ball.vx *= 600/sp;
    ball.vy *= 600/sp;
   }
  }
 });

 particles.forEach(p=>{
  p.x+=p.vx*dt;
  p.y+=p.vy*dt;
  p.vx*=.98;
  p.vy*=.98;
  p.t-=dt;
 });

 particles=particles.filter(p=>p.t>0);

 draw();
}

function draw() {
 ctx.clearRect(0,0,W,H);

 ctx.fillStyle="#1a4149";
 ctx.fillRect(0,0,W,H);

 for(let i=0;i<10;i++) {
  ctx.fillStyle=i%2 ? "#ffffff08" : "#00000008";
  ctx.fillRect(i*90,0,90,H);
 }

 ctx.strokeStyle="#b9e3e755";
 ctx.lineWidth=3;
 ctx.strokeRect(18,18,W-36,H-36);

 ctx.beginPath();
 ctx.moveTo(W/2,18);
 ctx.lineTo(W/2,H-18);
 ctx.stroke();

 ctx.beginPath();
 ctx.arc(W/2,H/2,80,0,7);
 ctx.stroke();

 ctx.fillStyle="#38bdf833";
 ctx.fillRect(0,H/2-76,26,152);

 ctx.fillStyle="#fb718533";
 ctx.fillRect(W-26,H/2-76,26,152);

 ctx.strokeStyle="#60d8ff";
 ctx.lineWidth=4;
 ctx.strokeRect(2,H/2-76,25,152);

 ctx.strokeStyle="#ff8678";
 ctx.strokeRect(W-27,H/2-76,25,152);

 ctx.fillStyle="#0b1225dd";
 ctx.fillRect(W/2-120,24,240,56);

 ctx.font="bold 28px system-ui";
 ctx.textAlign="center";

 ctx.fillStyle="#54d8ff";
 ctx.fillText(score[0],W/2-65,61);

 ctx.fillStyle="white";
 ctx.font="bold 18px system-ui";
 ctx.fillText(Math.ceil(time),W/2,59);

 ctx.fillStyle="#ff8276";
 ctx.font="bold 28px system-ui";
 ctx.fillText(score[1],W/2+65,61);

 ctx.textAlign="left";

 cars.forEach((p,i)=>{
  ctx.save();
  ctx.translate(p.x*24+450,p.y*24+255);
  ctx.rotate(p.a);

  ctx.fillStyle="#101827";
  ctx.fillRect(-15,-13,10,6);
  ctx.fillRect(7,-13,10,6);
  ctx.fillRect(-15,8,10,6);
  ctx.fillRect(7,8,10,6);

  ctx.fillStyle=p.color;
  ctx.beginPath();
  ctx.moveTo(21,0);
  ctx.lineTo(10,-12);
  ctx.lineTo(-13,-11);
  ctx.lineTo(-18,-5);
  ctx.lineTo(-18,5);
  ctx.lineTo(-13,11);
  ctx.lineTo(10,12);
  ctx.fill();

  ctx.fillStyle="#e6f8ff";
  ctx.fillRect(-5,-7,11,14);

  if(p.stun>0) {
   ctx.strokeStyle="#ffe8a3";
   ctx.beginPath();
   ctx.arc(0,0,24,0,7);
   ctx.stroke();
  }

  ctx.restore();
 });

 ctx.fillStyle="#e4edf3";
 ctx.beginPath();
 ctx.arc(ball.x*24+450,ball.y*24+255,ball.r,0,7);
 ctx.fill();

 ctx.strokeStyle="#627887";
 ctx.lineWidth=2;
 ctx.stroke();

 particles.forEach(p=>{
  ctx.globalAlpha=clamp(p.t,0,1);
  ctx.fillStyle=p.col;
  ctx.beginPath();
  ctx.arc(p.x,p.y,4,0,7);
  ctx.fill();
  ctx.globalAlpha=1;
 });

 ctx.fillStyle="#0008";
 ctx.fillRect(30,H-29,130,12);

 ctx.fillStyle="#3bcaff";
 ctx.fillRect(30,H-29,130*cars[0].boost/100,12);

 document.getElementById("boost").textContent =
 "P1 Boost: "+Math.round(cars[0].boost)+"%";
}

reset();
draw();

function frame(now) {
 let dt=Math.min(.033,(now-last)/1000||0);
 last=now;
 update(dt);
 requestAnimationFrame(frame);
}

requestAnimationFrame(frame);
})();
</script>
</body>
</html>
