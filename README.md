# cockpit2
<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>Motorfest Cockpit</title>
<script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"></script>
<style>
  :root {
    --bg: #0b0d12;
    --needle: #ff3b3b;
    --scale: #ffffff;
    --number: #ffffff;
    --ring: #2a2f3a;
    --mapring: #00e5ff;
  }
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body { margin: 0; height: 100%; background: var(--bg); color: #fff; font-family: system-ui, sans-serif; overflow: hidden; }
  .hidden { display: none !important; }
  button { background: #1d2230; color: #fff; border: 1px solid #39405a; border-radius: 10px; padding: 10px 14px; font-size: 16px; }
  button.primary { background: #0a84ff; border-color: #0a84ff; }
  button:disabled { opacity: .5; }

  /* ---------- Setup ---------- */
  #setup { position: fixed; inset: 0; display: flex; flex-direction: column; background: #000; }
  #stage { position: relative; flex: 1; min-height: 0; display: flex; align-items: center; justify-content: center; touch-action: none; }
  #video { max-width: 100%; max-height: 100%; }
  .box { position: absolute; border: 3px solid; border-radius: 6px; touch-action: none; }
  .box span { position: absolute; top: -26px; left: 0; font-size: 14px; padding: 2px 8px; border-radius: 6px; color: #000; font-weight: 600; white-space: nowrap; }
  .box .handle { position: absolute; right: -14px; bottom: -14px; width: 28px; height: 28px; border-radius: 50%; background: #fff; border: 3px solid currentColor; }
  #boxSpeed { border-color: #ffd60a; color: #ffd60a; }
  #boxSpeed span { background: #ffd60a; }
  #boxMap { border-color: #30d158; color: #30d158; border-radius: 50%; }
  #boxMap span { background: #30d158; left: 50%; transform: translateX(-50%); }
  #setupBar { padding: 10px; display: flex; gap: 8px; flex-wrap: wrap; justify-content: center; background: #111; }
  #detectMsg { position: absolute; left: 0; right: 0; top: 0; padding: 10px; text-align: center; background: rgba(0,0,0,.7); font-size: 15px; z-index: 3; }

  /* ---------- Cockpit ---------- */
  #cockpit { position: fixed; inset: 0; display: flex; align-items: center; justify-content: space-evenly; gap: 2vmin; padding: 2vmin; background: var(--bg); }
  @media (orientation: portrait) { #cockpit { flex-direction: column; } }
  #dialWrap { position: relative; width: min(60vmin, 90vw); aspect-ratio: 1; }
  #dial { width: 100%; height: 100%; }
  #speedText { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; color: var(--number); pointer-events: none; }
  #speedNum { font-size: 14vmin; font-weight: 800; line-height: 1; font-variant-numeric: tabular-nums; }
  #unitText { font-size: 3.5vmin; opacity: .7; margin-top: .5vmin; }
  #mapWrap { width: min(40vmin, 60vw); aspect-ratio: 1; border-radius: 50%; border: 1.2vmin solid var(--mapring); overflow: hidden; position: relative; box-shadow: 0 0 3vmin color-mix(in srgb, var(--mapring) 50%, transparent); }
  #mapCanvas { width: 100%; height: 100%; display: block; }
  #gear { position: fixed; top: 10px; right: 10px; opacity: .5; padding: 8px 12px; z-index: 5; }
  #fs { position: fixed; top: 10px; right: 70px; opacity: .5; padding: 8px 12px; z-index: 5; }
  #status { position: fixed; bottom: 8px; left: 10px; font-size: 12px; opacity: .5; }

  /* ---------- Einstellungen ---------- */
  #settings { position: fixed; inset: 0; background: rgba(0,0,0,.92); overflow-y: auto; padding: 16px; z-index: 10; }
  #settings h2 { margin-top: 0; }
  .row { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 10px 0; border-bottom: 1px solid #222; }
  .row input[type=color] { width: 56px; height: 36px; border: none; background: none; padding: 0; }
  .row input[type=range] { width: 55%; }
  .row input[type=number], .row select { background: #1d2230; color: #fff; border: 1px solid #39405a; border-radius: 8px; padding: 8px; font-size: 16px; width: 110px; }
  .group { margin: 18px 0 4px; font-weight: 700; color: #0a84ff; }
</style>
</head>
<body>

<!-- ======= SETUP ======= -->
<div id="setup">
  <div id="stage">
    <video id="video" playsinline muted autoplay></video>
    <div class="box" id="boxSpeed"><span>Geschwindigkeit</span><div class="handle"></div></div>
    <div class="box" id="boxMap"><span>Minimap</span><div class="handle"></div></div>
    <div id="detectMsg" class="hidden"></div>
  </div>
  <div id="setupBar">
    <button id="btnCam">Kamera wechseln</button>
    <button id="btnDetect">Automatisch erkennen</button>
    <button class="primary" id="btnStart">Starten</button>
  </div>
</div>

<!-- ======= COCKPIT ======= -->
<div id="cockpit" class="hidden">
  <div id="dialWrap">
    <svg id="dial" viewBox="0 0 200 200"></svg>
    <div id="speedText"><div id="speedNum">0</div><div id="unitText">km/h</div></div>
  </div>
  <div id="mapWrap"><canvas id="mapCanvas" width="400" height="400"></canvas></div>
  <button id="fs">⛶</button>
  <button id="gear">⚙</button>
  <div id="status"></div>
</div>

<!-- ======= EINSTELLUNGEN ======= -->
<div id="settings" class="hidden">
  <h2>Einstellungen</h2>

  <div class="group">Farben</div>
  <div class="row"><label>Hintergrund</label><input type="color" id="cBg"></div>
  <div class="row"><label>Nadel</label><input type="color" id="cNeedle"></div>
  <div class="row"><label>Skala</label><input type="color" id="cScale"></div>
  <div class="row"><label>Zahl</label><input type="color" id="cNumber"></div>
  <div class="row"><label>Tacho-Ring</label><input type="color" id="cRing"></div>
  <div class="row"><label>Minimap-Rahmen</label><input type="color" id="cMapRing"></div>

  <div class="group">Tacho</div>
  <div class="row"><label>Einheit</label>
    <select id="unit"><option value="km/h">km/h</option><option value="mph">mph</option></select></div>
  <div class="row"><label>Maximalwert</label><input type="number" id="maxSpeed" min="100" max="600" step="10"></div>
  <div class="row"><label>Glättung</label><input type="range" id="smooth" min="1" max="30"></div>

  <div class="group">Minimap-Filter</div>
  <div class="row"><label>Helligkeit</label><input type="range" id="fBright" min="50" max="200"></div>
  <div class="row"><label>Kontrast</label><input type="range" id="fContrast" min="50" max="250"></div>
  <div class="row"><label>Sättigung</label><input type="range" id="fSat" min="0" max="300"></div>
  <div class="row"><label>Farbton</label><input type="range" id="fHue" min="0" max="360"></div>
  <div class="row"><label>Zoom</label><input type="range" id="fZoom" min="100" max="300"></div>

  <div style="display:flex; gap:8px; margin-top:20px; flex-wrap:wrap;">
    <button class="primary" id="btnClose">Fertig</button>
    <button id="btnSetup">Bereiche neu wählen</button>
    <button id="btnReset">Zurücksetzen</button>
  </div>
</div>

<canvas id="grab" class="hidden"></canvas>
<canvas id="scan" class="hidden"></canvas>

<script>
/* ================= Einstellungen ================= */
const DEFAULTS = {
  cBg:'#0b0d12', cNeedle:'#ff3b3b', cScale:'#ffffff', cNumber:'#ffffff', cRing:'#2a2f3a', cMapRing:'#00e5ff',
  unit:'km/h', maxSpeed:360, smooth:12,
  fBright:100, fContrast:110, fSat:110, fHue:0, fZoom:100,
  speedBox:{x:.75,y:.80,w:.18,h:.10}, mapBox:{x:.03,y:.60,w:.20,h:.35}
};
let S = Object.assign({}, DEFAULTS, JSON.parse(localStorage.getItem('mfCockpit') || '{}'));
const save = () => localStorage.setItem('mfCockpit', JSON.stringify(S));

const $ = id => document.getElementById(id);
const video = $('video');
let stream = null, facing = 'environment';

/* ================= Kamera ================= */
async function startCam() {
  if (stream) stream.getTracks().forEach(t => t.stop());
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: facing, width: {ideal: 1920}, height: {ideal: 1080} }, audio: false
    });
    video.srcObject = stream;
    await video.play();
  } catch (e) {
    alert('Kamera nicht verfügbar. Läuft die Seite über HTTPS? Fehler: ' + e.message);
  }
}
$('btnCam').onclick = () => { facing = facing === 'environment' ? 'user' : 'environment'; startCam(); };

/* ================= Setup: Rahmen ziehen ================= */
const stage = $('stage');
function videoRect() {
  const r = video.getBoundingClientRect(), s = stage.getBoundingClientRect();
  return { x: r.left - s.left, y: r.top - s.top, w: r.width, h: r.height };
}
function layoutBoxes() {
  const v = videoRect();
  for (const [id, key] of [['boxSpeed','speedBox'], ['boxMap','mapBox']]) {
    const b = $(id), p = S[key];
    b.style.left = (v.x + p.x * v.w) + 'px';
    b.style.top = (v.y + p.y * v.h) + 'px';
    b.style.width = (p.w * v.w) + 'px';
    b.style.height = (p.h * v.h) + 'px';
  }
}
function makeDraggable(id, key, keepSquare) {
  const b = $(id), h = b.querySelector('.handle');
  let mode = null, sx, sy, start;
  const down = (e, m) => {
    e.preventDefault(); e.stopPropagation();
    mode = m; sx = e.clientX; sy = e.clientY; start = { ...S[key] };
    b.setPointerCapture?.(e.pointerId);
  };
  b.addEventListener('pointerdown', e => down(e, 'move'));
  h.addEventListener('pointerdown', e => down(e, 'size'));
  window.addEventListener('pointermove', e => {
    if (!mode) return;
    const v = videoRect();
    const dx = (e.clientX - sx) / v.w, dy = (e.clientY - sy) / v.h;
    const p = S[key];
    if (mode === 'move') {
      p.x = Math.min(Math.max(0, start.x + dx), 1 - p.w);
      p.y = Math.min(Math.max(0, start.y + dy), 1 - p.h);
    } else {
      p.w = Math.min(Math.max(.03, start.w + dx), 1 - p.x);
      p.h = Math.min(Math.max(.03, start.h + dy), 1 - p.y);
      if (keepSquare) {
        const px = Math.min(p.w * v.w, p.h * v.h);
        p.w = px / v.w; p.h = px / v.h;
      }
    }
    layoutBoxes();
  });
  window.addEventListener('pointerup', () => { if (mode) { mode = null; save(); } });
}
makeDraggable('boxSpeed', 'speedBox', false);
makeDraggable('boxMap', 'mapBox', true);
video.addEventListener('loadedmetadata', layoutBoxes);
window.addEventListener('resize', layoutBoxes);

/* ================= AUTO-ERKENNUNG ================= */
const scan = $('scan'), sctx = scan.getContext('2d', { willReadFrequently: true });
const msg = t => { const m = $('detectMsg'); m.textContent = t; m.classList.toggle('hidden', !t); };

// Bild der Kamera in ein Arbeitsbild kopieren (max. 1280 px breit)
function snapshot(maxW = 1280) {
  const k = Math.min(1, maxW / video.videoWidth);
  scan.width = Math.round(video.videoWidth * k);
  scan.height = Math.round(video.videoHeight * k);
  sctx.filter = 'none';
  sctx.drawImage(video, 0, 0, scan.width, scan.height);
  return sctx.getImageData(0, 0, scan.width, scan.height);
}

/* --- Minimap: Kreis per Hough-Transformation auf Kanten finden --- */
function detectCircle() {
  const W = 320, k = W / video.videoWidth, H = Math.round(video.videoHeight * k);
  const c = document.createElement('canvas'); c.width = W; c.height = H;
  const cx = c.getContext('2d', { willReadFrequently: true });
  cx.filter = 'grayscale(1) blur(1px)';
  cx.drawImage(video, 0, 0, W, H);
  const d = cx.getImageData(0, 0, W, H).data;
  const g = new Float32Array(W * H);
  for (let i = 0; i < W * H; i++) g[i] = d[i * 4];

  // Sobel-Kanten
  const edges = [];
  for (let y = 1; y < H - 1; y++) for (let x = 1; x < W - 1; x++) {
    const i = y * W + x;
    const gx = -g[i-W-1] - 2*g[i-1] - g[i+W-1] + g[i-W+1] + 2*g[i+1] + g[i+W+1];
    const gy = -g[i-W-1] - 2*g[i-W] - g[i-W+1] + g[i+W-1] + 2*g[i+W] + g[i+W+1];
    const m = Math.hypot(gx, gy);
    if (m > 90) edges.push([x, y, Math.atan2(gy, gx)]);
  }
  if (edges.length < 50) return null;

  // Hough: jede Kante stimmt entlang ihrer Richtung für mögliche Mittelpunkte ab
  const rMin = Math.round(W * 0.04), rMax = Math.round(W * 0.22);
  let best = null;
  for (let r = rMin; r <= rMax; r += 2) {
    const acc = new Uint16Array(W * H);
    for (const [x, y, a] of edges) {
      for (const s of [1, -1]) {
        const px = Math.round(x + s * r * Math.cos(a)), py = Math.round(y + s * r * Math.sin(a));
        if (px >= 0 && px < W && py >= 0 && py < H) acc[py * W + px]++;
      }
    }
    for (let i = 0; i < acc.length; i++) {
      // Normieren auf Umfang: größere Kreise brauchen mehr Stimmen
      const score = acc[i] / (2 * Math.PI * r);
      if (!best || score > best.score) best = { score, x: i % W, y: (i / W) | 0, r };
    }
  }
  if (!best || best.score < 0.22) return null;
  // zurück in Anteile des Videobilds
  const rx = best.r / W, ry = best.r / H;
  return { x: best.x / W - rx, y: best.y / H - ry, w: 2 * rx, h: 2 * ry, score: best.score };
}

/* --- Geschwindigkeit: ganze Bildfläche per OCR absuchen, Zahl mit Geschwindigkeits-Verhalten wählen --- */
async function detectSpeed(worker, frames) {
  const tracks = new Map(); // Position (gerastert) -> Liste von { n, box }
  for (let f = 0; f < frames; f++) {
    snapshot(1280);
    const { data } = await worker.recognize(scan);
    for (const w of (data.words || [])) {
      const t = (w.text || '').replace(/\D/g, '');
      if (!t || t.length > 3 || w.confidence < 50) continue;
      const b = w.bbox;
      const key = Math.round((b.x0 + b.x1) / 2 / 40) + ',' + Math.round((b.y0 + b.y1) / 2 / 40);
      if (!tracks.has(key)) tracks.set(key, []);
      tracks.get(key).push({ n: parseInt(t, 10), bbox: b, h: b.y1 - b.y0 });
    }
    msg('Suche Geschwindigkeit… ' + (f + 1) + '/' + frames);
    await new Promise(r => setTimeout(r, 450)); // Spiel weiterlaufen lassen, Werte ändern sich
  }
  // Bewertung: Zahl muss sich ändern, plausibel sein (0–400), hoch genug im Bild erkennbar sein
  let best = null;
  for (const [, arr] of tracks) {
    if (arr.length < Math.ceil(frames / 2)) continue;
    const vals = arr.map(a => a.n);
    const distinct = new Set(vals).size;
    const maxV = Math.max(...vals);
    if (maxV > 450) continue;
    const avgH = arr.reduce((s, a) => s + a.h, 0) / arr.length;
    const score = distinct * 3 + avgH / 10 + arr.length;
    if (distinct >= 2 && (!best || score > best.score)) best = { score, bbox: arr[arr.length - 1].bbox };
  }
  if (!best) return null;
  const k = scan.width, kh = scan.height, b = best.bbox, padX = (b.x1 - b.x0) * 0.25, padY = (b.y1 - b.y0) * 0.25;
  return {
    x: Math.max(0, (b.x0 - padX) / k), y: Math.max(0, (b.y0 - padY) / kh),
    w: Math.min(1, (b.x1 - b.x0 + 2 * padX) / k), h: Math.min(1, (b.y1 - b.y0 + 2 * padY) / kh)
  };
}

$('btnDetect').onclick = async () => {
  const btn = $('btnDetect'); btn.disabled = true;
  try {
    if (!video.videoWidth) { msg('Kamera läuft noch nicht.'); return; }
    msg('Suche Minimap…');
    await new Promise(r => setTimeout(r, 50));
    const circle = detectCircle();
    if (circle) { S.mapBox = { x: circle.x, y: circle.y, w: circle.w, h: circle.h }; layoutBoxes(); }

    msg('Lade Texterkennung…');
    if (!worker) await initOCR();
    await worker.setParameters({ tessedit_char_whitelist: '0123456789', tessedit_pageseg_mode: '11' });
    const sp = await detectSpeed(worker, 6);
    await worker.setParameters({ tessedit_char_whitelist: '0123456789', tessedit_pageseg_mode: '7' });
    if (sp) { S.speedBox = sp; layoutBoxes(); }

    save();
    const parts = [];
    parts.push(circle ? 'Minimap gefunden' : 'Minimap nicht gefunden');
    parts.push(sp ? 'Geschwindigkeit gefunden' : 'Geschwindigkeit nicht gefunden');
    msg(parts.join(' · ') + '. Rahmen bei Bedarf nachziehen.');
    setTimeout(() => msg(''), 6000);
  } catch (e) {
    msg('Fehler bei der Erkennung: ' + e.message);
    setTimeout(() => msg(''), 6000);
  } finally { btn.disabled = false; }
};

/* ================= Tacho zeichnen (SVG) ================= */
const dial = $('dial');
const START_ANGLE = 135, SWEEP = 270;
const pol = (a, r) => { const t = (a - 90) * Math.PI / 180; return [100 + r * Math.cos(t), 100 + r * Math.sin(t)]; };
function buildDial() {
  const max = S.maxSpeed, step = max <= 240 ? 20 : max <= 400 ? 40 : 50;
  let svg = '';
  const [x1, y1] = pol(START_ANGLE, 92), [x2, y2] = pol(START_ANGLE + SWEEP, 92);
  svg += `<path d="M${x1} ${y1} A92 92 0 1 1 ${x2} ${y2}" fill="none" stroke="var(--ring)" stroke-width="8" stroke-linecap="round"/>`;
  svg += `<path id="arc" fill="none" stroke="var(--needle)" stroke-width="8" stroke-linecap="round" opacity=".35"/>`;
  for (let v = 0; v <= max; v += step / 2) {
    const a = START_ANGLE + (v / max) * SWEEP, major = v % step === 0;
    const [ax, ay] = pol(a, major ? 78 : 83), [bx, by] = pol(a, 88);
    svg += `<line x1="${ax}" y1="${ay}" x2="${bx}" y2="${by}" stroke="var(--scale)" stroke-width="${major ? 2 : 1}"/>`;
    if (major) {
      const [tx, ty] = pol(a, 66);
      svg += `<text x="${tx}" y="${ty}" fill="var(--scale)" font-size="9" text-anchor="middle" dominant-baseline="middle" font-family="system-ui">${v}</text>`;
    }
  }
  svg += `<g id="needle"><polygon points="98,100 102,100 100,18" fill="var(--needle)"/><circle cx="100" cy="100" r="6" fill="var(--needle)"/></g>`;
  dial.innerHTML = svg;
}
let shown = 0, target = 0;
function renderDial(v) {
  const frac = Math.min(Math.max(v / S.maxSpeed, 0), 1);
  $('needle').setAttribute('transform', `rotate(${START_ANGLE + frac * SWEEP} 100 100)`);
  const [sx, sy] = pol(START_ANGLE, 92), [ex, ey] = pol(START_ANGLE + frac * SWEEP, 92);
  $('arc').setAttribute('d', frac <= 0 ? '' : `M${sx} ${sy} A92 92 0 ${frac * SWEEP > 180 ? 1 : 0} 1 ${ex} ${ey}`);
  $('speedNum').textContent = Math.round(v);
}

/* ================= Farben und Filter anwenden ================= */
function applyStyle() {
  const r = document.documentElement.style;
  r.setProperty('--bg', S.cBg); r.setProperty('--needle', S.cNeedle); r.setProperty('--scale', S.cScale);
  r.setProperty('--number', S.cNumber); r.setProperty('--ring', S.cRing); r.setProperty('--mapring', S.cMapRing);
  $('unitText').textContent = S.unit;
  buildDial(); renderDial(shown);
}

/* ================= Hauptschleife: Minimap ================= */
const grab = $('grab'), gctx = grab.getContext('2d', { willReadFrequently: true });
const mapCanvas = $('mapCanvas'), mctx = mapCanvas.getContext('2d');
let running = false;
function drawMap() {
  if (!running) return;
  const vw = video.videoWidth, vh = video.videoHeight;
  if (vw) {
    const b = S.mapBox, z = S.fZoom / 100;
    let sw = b.w * vw / z, sh = b.h * vh / z;
    let sx = b.x * vw + (b.w * vw - sw) / 2, sy = b.y * vh + (b.h * vh - sh) / 2;
    mctx.filter = `brightness(${S.fBright}%) contrast(${S.fContrast}%) saturate(${S.fSat}%) hue-rotate(${S.fHue}deg)`;
    mctx.drawImage(video, sx, sy, sw, sh, 0, 0, mapCanvas.width, mapCanvas.height);
  }
  requestAnimationFrame(drawMap);
}

/* ================= OCR: Geschwindigkeit ================= */
let worker = null, lastGood = 0;
async function initOCR() {
  worker = await Tesseract.createWorker('eng');
  await worker.setParameters({ tessedit_char_whitelist: '0123456789', tessedit_pageseg_mode: '7' });
}
function preprocessSpeed() {
  const vw = video.videoWidth, vh = video.videoHeight, b = S.speedBox;
  const sx = b.x * vw, sy = b.y * vh, sw = b.w * vw, sh = b.h * vh;
  const scale = Math.max(1, 120 / sh);
  grab.width = Math.round(sw * scale); grab.height = Math.round(sh * scale);
  gctx.filter = 'grayscale(1) contrast(180%)';
  gctx.drawImage(video, sx, sy, sw, sh, 0, 0, grab.width, grab.height);
  const img = gctx.getImageData(0, 0, grab.width, grab.height), d = img.data;
  let sum = 0; for (let i = 0; i < d.length; i += 4) sum += d[i];
  const thr = sum / (d.length / 4);
  for (let i = 0; i < d.length; i += 4) { const v = d[i] > thr ? 0 : 255; d[i] = d[i+1] = d[i+2] = v; }
  gctx.putImageData(img, 0, 0);
}
async function ocrLoop() {
  while (running) {
    try {
      if (video.videoWidth && worker) {
        preprocessSpeed();
        const { data } = await worker.recognize(grab);
        const txt = (data.text || '').replace(/\D/g, '');
        if (txt.length >= 1 && txt.length <= 3) {
          const n = parseInt(txt, 10);
          if (n <= S.maxSpeed && Math.abs(n - lastGood) < 120) { target = n; lastGood = n; }
          else if (Math.abs(n - lastGood) >= 120) { lastGood = (lastGood + n) / 2; }
        }
        $('status').textContent = 'Erkannt: ' + (txt || '–');
      }
    } catch (e) { $('status').textContent = 'OCR-Fehler'; }
    await new Promise(r => setTimeout(r, 30));
  }
}
function smoothLoop() {
  if (!running) return;
  const k = 1 / Math.max(1, S.smooth);
  shown += (target - shown) * k;
  if (Math.abs(target - shown) < .2) shown = target;
  renderDial(shown);
  requestAnimationFrame(smoothLoop);
}

/* ================= Start / Stop ================= */
async function startCockpit() {
  $('setup').classList.add('hidden');
  $('cockpit').classList.remove('hidden');
  applyStyle();
  running = true;
  $('status').textContent = 'Lade Texterkennung…';
  if (!worker) await initOCR();
  $('status').textContent = '';
  drawMap(); smoothLoop(); ocrLoop();
}
$('btnStart').onclick = startCockpit;

/* ================= Einstellungsmenü ================= */
const inputs = ['cBg','cNeedle','cScale','cNumber','cRing','cMapRing','unit','maxSpeed','smooth','fBright','fContrast','fSat','fHue','fZoom'];
function fillInputs() { inputs.forEach(k => { $(k).value = S[k]; }); }
inputs.forEach(k => {
  $(k).addEventListener('input', () => {
    S[k] = ($(k).type === 'number' || $(k).type === 'range') ? Number($(k).value) : $(k).value;
    save(); applyStyle();
  });
});
$('gear').onclick = () => { fillInputs(); $('settings').classList.remove('hidden'); };
$('btnClose').onclick = () => $('settings').classList.add('hidden');
$('btnSetup').onclick = () => {
  running = false;
  $('settings').classList.add('hidden'); $('cockpit').classList.add('hidden'); $('setup').classList.remove('hidden');
  layoutBoxes();
};
$('btnReset').onclick = () => { if (confirm('Alles zurücksetzen?')) { S = JSON.parse(JSON.stringify(DEFAULTS)); save(); fillInputs(); applyStyle(); layoutBoxes(); } };
$('fs').onclick = () => { document.fullscreenElement ? document.exitFullscreen() : document.documentElement.requestFullscreen?.(); };

/* ================= Los geht's ================= */
applyStyle();
startCam().then(layoutBoxes);
</script>
</body>
</html>
