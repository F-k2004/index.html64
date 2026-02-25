<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>🌞 Solar Storm – Frequency Dependent Noise</title>
<style>
html,body{margin:0;overflow:hidden;background:#000014;font-family:system-ui}
canvas{display:block}
.hud{
 position:absolute;left:16px;top:16px;
 padding:14px 18px;border-radius:14px;
 background:rgba(255,255,255,0.06);
 backdrop-filter:blur(10px);
 color:#d9f3ff;font-size:13px;min-width:520px;
}
.good{color:#9ff0ff}
.md{color:#ffd29f}
.bad{color:#ff9f9f}
.storm{color:#ffcc66}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

const AU=65;
const sun={x:w/2,y:h/2};
const earth={a:AU*1,angle:0};
const ship={distance:AU*1,angle:0.4,velocity:0.22};

const c_light=3e8;
const txPower=20;
const dishDiameter=3.7;

const bands={
 X:{freq:8.4e9},
 Ka:{freq:32e9}
};

let stormLevel=0;
let stormTimer=0;

function erfc(x){
 const z=Math.abs(x);
 const t=1/(1+0.5*z);
 const r=t*Math.exp(-z*z-1.26551223+
   t*(1.00002368+
   t*(0.37409196+
   t*(0.09678418+
   t*(-0.18628806+
   t*(0.27886807+
   t*(-1.13520398+
   t*(1.48851587+
   t*(-0.82215223+
   t*0.17087277)))))))));
 return x>=0 ? r : 2-r;
}

function computeBER(snr){
 return 0.5*erfc(Math.sqrt(snr));
}

function solarNoise(freq){
 const alpha=2.5;
 const K=5e-20;
 return K*stormLevel*Math.pow(freq,-alpha);
}

function link(distAU,band){
 const f=band.freq;
 const lambda=c_light/f;
 const distMeters=distAU*1.496e11;

 const fspl=(4*Math.PI*distMeters*f/c_light)**2;
 const gain=(Math.PI*dishDiameter/lambda)**2;
 const pr=(txPower*gain)/fspl;

 const baseNoise=1e-21;
 const stormNoise=solarNoise(f);
 const noise=baseNoise+stormNoise;

 const snr=pr/noise;
 return {snr,ber:computeBER(snr)};
}

function update(){
 earth.angle+=0.0012;
 ship.distance+=ship.velocity;
 ship.angle+=0.002;

 // Random storm trigger
 stormTimer++;
 if(stormTimer>600 && Math.random()<0.002){
   stormLevel=1;
   stormTimer=0;
 }
 stormLevel*=0.995; // decay
}

function pos(a,ang){
 return {x:sun.x+Math.cos(ang)*a,y:sun.y+Math.sin(ang)*a};
}

function draw(){
 ctx.fillStyle="rgba(0,0,20,0.4)";
 ctx.fillRect(0,0,w,h);

 update();

 const pe=pos(earth.a,earth.angle);
 const pship={
  x:sun.x+Math.cos(ship.angle)*ship.distance,
  y:sun.y+Math.sin(ship.angle)*ship.distance
 };

 ctx.fillStyle=stormLevel>0.2?"#ff8844":"#ffcc66";
 ctx.beginPath();ctx.arc(sun.x,sun.y,12,0,Math.PI*2);ctx.fill();

 ctx.fillStyle="#2a7fff";
 ctx.beginPath();ctx.arc(pe.x,pe.y,6,0,Math.PI*2);ctx.fill();

 ctx.fillStyle="#fff";
 ctx.beginPath();ctx.arc(pship.x,pship.y,4,0,Math.PI*2);ctx.fill();

 const distAU=Math.hypot(pship.x-pe.x,pship.y-pe.y)/AU;

 const xlink=link(distAU,bands.X);
 const kalink=link(distAU,bands.Ka);

 const best=xlink.ber<kalink.ber?"X":"Ka";

 document.getElementById("hud").innerHTML=`
🌞 Solar Storm Level: <span class="storm">${stormLevel.toFixed(2)}</span><br>
📏 Distance: ${distAU.toFixed(1)} AU<br><br>

📡 X-band<br>
SNR: ${xlink.snr.toExponential(2)}<br>
BER: ${xlink.ber.toExponential(2)}<br><br>

📡 Ka-band<br>
SNR: ${kalink.snr.toExponential(2)}<br>
BER: ${kalink.ber.toExponential(2)}<br><br>

🤖 AI Selected: <b>${best}-band</b>
`;

 requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
