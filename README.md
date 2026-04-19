<!DOCTYPE html>

<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Adam – GitHub Profile</title>
<link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;700&family=Syne:wght@700;800&display=swap" rel="stylesheet">
<style>
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
–bg: #0d1117;
–surface: #161b22;
–border: #30363d;
–text: #e6edf3;
–muted: #7d8590;
–green: #39d353;
–cyan: #58a6ff;
–purple: #bc8cff;
–orange: #ffa657;
}

body {
background: var(–bg);
color: var(–text);
font-family: ‘JetBrains Mono’, monospace;
display: flex;
justify-content: center;
padding: 40px 16px;
min-height: 100vh;
}

.card {
width: 100%;
max-width: 720px;
background: var(–surface);
border: 1px solid var(–border);
border-radius: 12px;
overflow: hidden;
}

/* ── GIF BANNER ── */
.banner {
position: relative;
width: 100%;
height: 200px;
background: #000;
overflow: hidden;
}

/* Starfield */
.stars {
position: absolute; inset: 0;
background:
radial-gradient(1px 1px at 10% 20%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 30% 70%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 50% 10%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 70% 50%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 90% 30%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 15% 85%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 45% 45%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 80% 80%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 60% 25%, #fff 0%, transparent 100%),
radial-gradient(1px 1px at 25% 55%, #fff 0%, transparent 100%),
radial-gradient(1.5px 1.5px at 55% 65%, rgba(255,255,255,0.6) 0%, transparent 100%),
radial-gradient(1.5px 1.5px at 85% 15%, rgba(255,255,255,0.6) 0%, transparent 100%),
radial-gradient(1.5px 1.5px at 5% 40%, rgba(255,255,255,0.6) 0%, transparent 100%);
animation: twinkle 4s ease-in-out infinite alternate;
}
@keyframes twinkle {
0%   { opacity: 0.6; }
100% { opacity: 1; }
}

/* Shooting star */
.shoot {
position: absolute;
top: 30px; left: -60px;
width: 60px; height: 2px;
background: linear-gradient(90deg, transparent, #fff);
border-radius: 2px;
animation: shoot 3s linear infinite;
opacity: 0;
}
.shoot:nth-child(2) { top: 80px; animation-delay: 1.5s; }
.shoot:nth-child(3) { top: 140px; animation-delay: 2.8s; }
@keyframes shoot {
0%   { left: -60px; opacity: 0; }
10%  { opacity: 1; }
60%  { opacity: 1; }
100% { left: 110%; opacity: 0; }
}

/* Orbit rings */
.orbit-wrap {
position: absolute;
top: 50%; left: 50%;
transform: translate(-50%, -50%);
}
.ring {
position: absolute;
border: 1px solid rgba(88,166,255,0.15);
border-radius: 50%;
transform: translate(-50%, -50%);
}
.ring-1 { width: 120px; height: 80px; top: 0; left: 0; animation: spin1 8s linear infinite; }
.ring-2 { width: 180px; height: 120px; top: 0; left: 0; animation: spin1 12s linear infinite reverse; border-color: rgba(188,140,255,0.12); }
.ring-3 { width: 250px; height: 160px; top: 0; left: 0; animation: spin1 18s linear infinite; border-color: rgba(57,211,83,0.1); }
@keyframes spin1 { to { transform: translate(-50%,-50%) rotate(360deg); } }

/* Planet */
.planet {
position: absolute;
top: 50%; left: 50%;
transform: translate(-50%, -50%);
width: 48px; height: 48px;
border-radius: 50%;
background: radial-gradient(circle at 35% 35%, #58a6ff, #1a56a0);
box-shadow: 0 0 20px rgba(88,166,255,0.5), 0 0 60px rgba(88,166,255,0.2);
animation: pulse 3s ease-in-out infinite;
}
@keyframes pulse {
0%,100% { box-shadow: 0 0 20px rgba(88,166,255,0.5), 0 0 60px rgba(88,166,255,0.2); }
50%      { box-shadow: 0 0 30px rgba(88,166,255,0.8), 0 0 80px rgba(88,166,255,0.4); }
}

/* Floating code bits */
.code-bit {
position: absolute;
font-size: 11px;
color: rgba(57,211,83,0.5);
white-space: nowrap;
animation: floatUp 6s linear infinite;
opacity: 0;
}
.code-bit:nth-child(1)  { left: 5%;  animation-delay: 0s;   }
.code-bit:nth-child(2)  { left: 15%; animation-delay: 1s;   }
.code-bit:nth-child(3)  { left: 70%; animation-delay: 0.5s; color: rgba(188,140,255,0.5); }
.code-bit:nth-child(4)  { left: 85%; animation-delay: 2s;   color: rgba(255,166,87,0.5); }
.code-bit:nth-child(5)  { left: 40%; animation-delay: 3s;   color: rgba(88,166,255,0.5); }
@keyframes floatUp {
0%   { bottom: -20px; opacity: 0; }
10%  { opacity: 1; }
90%  { opacity: 0.6; }
100% { bottom: 220px; opacity: 0; }
}

/* Banner label */
.banner-label {
position: absolute;
bottom: 12px; right: 16px;
font-size: 10px;
color: var(–muted);
letter-spacing: 2px;
text-transform: uppercase;
}

/* ── CONTENT ── */
.content {
padding: 28px 32px 32px;
}

.greeting {
font-family: ‘Syne’, sans-serif;
font-size: 28px;
font-weight: 800;
color: var(–text);
margin-bottom: 4px;
}
.greeting span { color: var(–cyan); }

.tag-line {
font-size: 12px;
color: var(–muted);
margin-bottom: 20px;
letter-spacing: 1px;
}

.divider {
border: none;
border-top: 1px solid var(–border);
margin: 20px 0;
}

.bio {
font-size: 13px;
line-height: 1.8;
color: #c9d1d9;
margin-bottom: 4px;
}
.bio .highlight { color: var(–green); }
.bio .highlight2 { color: var(–purple); }

.rocket {
display: inline-block;
animation: rocketBounce 1s ease-in-out infinite alternate;
}
@keyframes rocketBounce {
from { transform: translateY(0); }
to   { transform: translateY(-4px); }
}

hr { border: none; border-top: 1px solid var(–border); margin: 20px 0; }

.section-title {
font-size: 11px;
letter-spacing: 3px;
text-transform: uppercase;
color: var(–muted);
margin-bottom: 16px;
}

.skills-grid {
display: flex;
flex-wrap: wrap;
gap: 10px;
justify-content: center;
}

.skill-chip {
display: flex;
align-items: center;
gap: 7px;
background: rgba(88,166,255,0.06);
border: 1px solid rgba(88,166,255,0.2);
border-radius: 8px;
padding: 7px 14px;
font-size: 12px;
color: var(–cyan);
transition: all 0.2s;
cursor: default;
}
.skill-chip:hover {
background: rgba(88,166,255,0.14);
border-color: rgba(88,166,255,0.5);
transform: translateY(-2px);
}
.skill-chip.green  { color: var(–green);  background: rgba(57,211,83,0.06);   border-color: rgba(57,211,83,0.2);  }
.skill-chip.green:hover  { background: rgba(57,211,83,0.14);  border-color: rgba(57,211,83,0.5); }
.skill-chip.purple { color: var(–purple); background: rgba(188,140,255,0.06); border-color: rgba(188,140,255,0.2); }
.skill-chip.purple:hover { background: rgba(188,140,255,0.14); border-color: rgba(188,140,255,0.5); }
.skill-chip.orange { color: var(–orange); background: rgba(255,166,87,0.06);  border-color: rgba(255,166,87,0.2); }
.skill-chip.orange:hover { background: rgba(255,166,87,0.14); border-color: rgba(255,166,87,0.5); }

.skill-icon { font-size: 16px; }

.footer {
margin-top: 24px;
font-size: 11px;
color: var(–muted);
text-align: center;
letter-spacing: 1px;
}
.cursor {
display: inline-block;
width: 8px; height: 14px;
background: var(–green);
margin-left: 3px;
vertical-align: text-bottom;
animation: blink 1s step-end infinite;
}
@keyframes blink { 0%,100% { opacity:1 } 50% { opacity:0 } }
</style>

</head>
<body>
<div class="card">

  <!-- ANIMATED GIF BANNER -->

  <div class="banner">
    <div class="stars"></div>
    <div class="shoot"></div>
    <div class="shoot"></div>
    <div class="shoot"></div>

```
<div class="orbit-wrap">
  <div class="ring ring-1"></div>
  <div class="ring ring-2"></div>
  <div class="ring ring-3"></div>
  <div class="planet"></div>
</div>

<div class="code-bit">&lt;html&gt;</div>
<div class="code-bit">const dev = 🚀</div>
<div class="code-bit">npm start</div>
<div class="code-bit">{ fullstack }</div>
<div class="code-bit">async/await</div>

<div class="banner-label">fullstack.dev</div>
```

  </div>

  <!-- CONTENT -->

  <div class="content">
    <div class="greeting">Hi, I'm <span>Adam</span> 👋</div>
    <div class="tag-line">// beginner fullstack developer</div>

```
<p class="bio">
  I'm a <span class="highlight">beginner fullstack developer</span> who is currently learning
  how to build <span class="highlight2">modern web applications</span> step by step.
  <br><br>
  This GitHub is where I document my learning journey and progress as a fullstack developer <span class="rocket">🚀</span>
</p>

<hr>

<div class="section-title">🌐 Skills</div>
<div class="skills-grid">
  <div class="skill-chip"><span class="skill-icon">🌐</span> HTML</div>
  <div class="skill-chip"><span class="skill-icon">🎨</span> CSS</div>
  <div class="skill-chip orange"><span class="skill-icon">⚡</span> JavaScript</div>
  <div class="skill-chip purple"><span class="skill-icon">⚛️</span> React</div>
  <div class="skill-chip"><span class="skill-icon">🌊</span> Tailwind</div>
  <div class="skill-chip green"><span class="skill-icon">🟢</span> Node.js</div>
  <div class="skill-chip green"><span class="skill-icon">🛤️</span> Express</div>
  <div class="skill-chip green"><span class="skill-icon">🍃</span> MongoDB</div>
</div>

<div class="footer">
  learning in progress<span class="cursor"></span>
</div>
```

  </div>

</div>
</body>
</html>
