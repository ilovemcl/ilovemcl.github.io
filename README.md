<html lang="en">
<head>
<meta charset="UTF-8">
<title>Monte Carlo Localization</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #060910;
    color: #e2e8f0;
    font-family: 'Courier New', monospace;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    padding: 24px;
  }
  h1  { font-size: 22px; letter-spacing: 0.05em; color: #f1f5f9; margin-bottom: 4px; }
  .subtitle { font-size: 11px; letter-spacing: 0.3em; color: #64748b; text-transform: uppercase; margin-bottom: 6px; }
  .hint     { font-size: 11px; color: #38bdf8; margin-bottom: 18px; min-height: 16px; }
  .layout   { display: flex; gap: 20px; align-items: flex-start; flex-wrap: wrap; justify-content: center; }

  .canvas-wrap { position: relative; }
  canvas { border: 1px solid rgba(255,255,255,0.08); border-radius: 8px; display: block; }

  .legend { position: absolute; bottom: 48px; left: 8px; display: flex; flex-direction: column; gap: 4px; }
  .legend-item { display: flex; align-items: center; gap: 6px; font-size: 10px; color: #94a3b8; }
  .legend-dot  { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }

  .beam-warn { position: absolute; top: 8px; left: 50%; transform: translateX(-50%);
    background: rgba(251,146,60,0.15); border: 1px solid rgba(251,146,60,0.4);
    color: #fb923c; font-size: 10px; padding: 3px 10px; border-radius: 20px;
    white-space: nowrap; opacity: 0; transition: opacity 0.3s; pointer-events: none; }
  .beam-warn.show { opacity: 1; }

  .sidebar    { display: flex; flex-direction: column; gap: 12px; width: 210px; }
  .panel      { background: rgba(255,255,255,0.03); border: 1px solid rgba(255,255,255,0.07); border-radius: 8px; padding: 14px; }
  .panel-title{ font-size: 10px; color: #64748b; letter-spacing: 0.2em; margin-bottom: 10px; }
  .stat-row   { display: flex; justify-content: space-between; margin-bottom: 5px; font-size: 12px; }
  .stat-label { color: #64748b; }
  .stat-val   { color: #e2e8f0; font-weight: 600; }
  .stat-val.warn { color: #fb923c; }

  .btn      { padding: 10px; border-radius: 6px; border: none; cursor: pointer; font-size: 12px;
    font-family: 'Courier New', monospace; font-weight: 700; letter-spacing: 0.1em;
    width: 100%; transition: all 0.2s; }
  .btn-grey { background: rgba(255,255,255,0.04); color: #94a3b8; border: 1px solid rgba(255,255,255,0.1); }

  input[type=range] { width: 100%; accent-color: #38bdf8; margin-bottom: 10px; cursor: pointer; }
</style>
</head>
<body>

<div class="subtitle">Monte Carlo Localization — 4-Beam Pass-Through Sensors (Wall + Object)</div>
<h1>Monte Carlo Localization 500</h1>

<div class="layout">
  <div class="canvas-wrap">
    <canvas id="canvas" width="440" height="440"></canvas>
    <div class="beam-warn" id="beam-warn">⚠ All beams out of range — particles drifting</div>

    <div class="legend">
      <div class="legend-item"><div class="legend-dot" style="background:#f87171"></div>True robot</div>
      <div class="legend-item"><div class="legend-dot" style="background:#e2e8f0;opacity:0.5"></div>Particles</div>
      <div class="legend-item"><div class="legend-dot" style="background:#34d399"></div>Estimate</div>
      <div class="legend-item"><div class="legend-dot" style="background:#38bdf8"></div>Front beam (F)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#a78bfa"></div>Right beam (R)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#fb923c"></div>Back beam (B)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#34d399"></div>Left beam (L)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#fbbf24;border-radius:2px;"></div>Objects (0.5×0.5 ft)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#fbbf24;opacity:0.4;"></div>Object particles</div>
    </div>
  </div>

  <div class="sidebar">
    <div class="panel">
      <div class="panel-title">STATUS</div>
      <div class="stat-row"><span class="stat-label">Step</span><span class="stat-val" id="s-step">0</span></div>
      <div class="stat-row"><span class="stat-label">Error</span><span class="stat-val" id="s-error">—</span></div>
      <div class="stat-row"><span class="stat-label">Beams reading</span><span class="stat-val" id="s-beams">0 / 4</span></div>
      <div class="stat-row"><span class="stat-label">Beam width</span><span class="stat-val" id="s-fov">20°</span></div>
      <div class="stat-row"><span class="stat-label">Sensor range</span><span class="stat-val" id="s-range">72 in</span></div>
      <div class="stat-row"><span class="stat-label">Heading (gyro)</span><span class="stat-val" id="s-heading">0.0°</span></div>
      <div class="stat-row"><span class="stat-label">Robot X</span><span class="stat-val" id="s-rx">—</span></div>
      <div class="stat-row"><span class="stat-label">Robot Y</span><span class="stat-val" id="s-ry">—</span></div>
      <div class="stat-row"><span class="stat-label">Particles</span><span class="stat-val">500</span></div>
    </div>

    <div class="panel">
      <div class="panel-title">BEAM READINGS (wall | ◆obj)</div>
      <div class="stat-row"><span class="stat-label" style="color:#38bdf8">▶ Front</span><span class="stat-val" id="b-f">—</span></div>
      <div class="stat-row"><span class="stat-label" style="color:#a78bfa">▶ Right</span><span class="stat-val" id="b-r">—</span></div>
      <div class="stat-row"><span class="stat-label" style="color:#fb923c">▶ Back</span> <span class="stat-val" id="b-b">—</span></div>
      <div class="stat-row"><span class="stat-label" style="color:#34d399">▶ Left</span> <span class="stat-val" id="b-l">—</span></div>
    </div>

    <div class="panel">
      <div class="panel-title">ERROR HISTORY</div>
      <svg id="chart" width="182" height="55" style="display:block"></svg>
    </div>

    <div class="panel">
      <div class="panel-title">ROBOT ACCURACY</div>
      <div class="stat-row"><span class="stat-label">Current error</span><span class="stat-val" id="ra-cur">—</span></div>
      <div class="stat-row"><span class="stat-label">Best error</span><span class="stat-val" id="ra-best">—</span></div>
      <div class="stat-row"><span class="stat-label">Avg (last 20)</span><span class="stat-val" id="ra-avg">—</span></div>
      <div class="stat-row"><span class="stat-label">Converged</span><span class="stat-val" id="ra-conv">—</span></div>
    </div>

    <div class="panel">
      <div class="panel-title">SENSOR SETTINGS</div>
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Beam width</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-fov">20°</span>
      </div>
      <input type="range" id="sl-fov" min="2" max="45" value="20" step="1">
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Sensor range (in)</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-range">72</span>
      </div>
      <input type="range" id="sl-range" min="1" max="220" value="72" step="1">
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Sensor noise (in)</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-noise">0.2</span>
      </div>
      <input type="range" id="sl-noise" min="0.1" max="6" value="0.2" step="0.1" style="margin-bottom:10px;">
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Gyro drift (rad/step)</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-gyro">0.01</span>
      </div>
      <input type="range" id="sl-gyro" min="0.001" max="0.3" value="0.01" step="0.001" style="margin-bottom:0;">
    </div>

    <div style="display:flex;flex-direction:column;gap:8px;">
      <button class="btn btn-grey" id="btn-reset">↺ RESET</button>
    </div>

    <div class="panel">
      <div class="panel-title">OBJECT LOCALIZATION ACCURACY</div>
      <div id="obj-accuracy" style="display:flex;flex-direction:column;gap:4px;"></div>
      <div style="margin-top:8px;font-size:10px;color:#475569;">Particle cluster vs true object center</div>
    </div>

  </div>
</div>

<script>
// map set up
const ROOM_MM      = 3600;  // Room is 3600mm × 3600mm (≈ 12ft × 12ft = 144in × 144in)
const CANVAS_SIZE  = 440;
const MARGIN_PX    = 20;
const ROOM_PX      = 400;
const MM_PER_PX    = ROOM_MM / ROOM_PX;   // 9 mm per pixel
const PX_PER_MM    = ROOM_PX / ROOM_MM;

// Unit conversions
function mmToPx(mm)     { return mm * PX_PER_MM; }
function pxToMm(px)     { return px * MM_PER_PX; }
function inToMm(inches) { return inches * 25.4; }
function mmToIn(mm)     { return mm / 25.4; }
function pxToIn(px)     { return mmToIn(pxToMm(px)); }

// Room walls in canvas-pixel coords (top-left corner of room = MARGIN_PX, MARGIN_PX)
// The robot lives inside this box
const WALL = {
  left:   MARGIN_PX,
  right:  MARGIN_PX + ROOM_PX,
  top:    MARGIN_PX,
  bottom: MARGIN_PX + ROOM_PX,
};

const NUM_PARTICLES  = 500;
const MOTION_NOISE   = 0.3;   // Position noise on particle movement (px)
const RESAMPLE_NOISE = 0.6;   // Jitter added on resample (px)

// Controlled by sliders
let SENSOR_NOISE_IN  = 0.2;
let BEAM_WIDTH_DEG   = 20;
let SENSOR_RANGE_IN  = 72; // 6 ft
// Gyro drift: how much the gyro heading reading deviates from truth (radians std dev per step)
// 0 = perfect gyro, 0.01 = very good, 0.05 = mediocre, 0.2 = bad
let GYRO_NOISE       = 0.01;

// Internal mm values used for geometry
function sensorRangeMm()  { return inToMm(SENSOR_RANGE_IN); }
function sensorRangePx()  { return mmToPx(sensorRangeMm()); }
function sensorNoiseMm()  { return inToMm(SENSOR_NOISE_IN); }

// Four beams: front (0°), right (90°), back (180°), left (270°) relative to heading
const BEAM_OFFSETS = [0, Math.PI / 2, Math.PI, -Math.PI / 2];
const BEAM_LABELS  = ["F", "R", "B", "L"];
const BEAM_COLORS  = ["#38bdf8", "#a78bfa", "#fb923c", "#34d399"];
const BEAM_IDS     = ["b-f", "b-r", "b-b", "b-l"];

function gauss(mean, std) {
  let u = 0, v = 0;
  while (!u) u = Math.random();
  while (!v) v = Math.random();
  return mean + std * Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v);
}
function dist(x1,y1,x2,y2) { return Math.sqrt((x1-x2)**2+(y1-y2)**2); }
function clamp(v,lo,hi)     { return Math.max(lo,Math.min(hi,v)); }
function normAngle(a) {
  while (a >  Math.PI) a -= 2*Math.PI;
  while (a < -Math.PI) a += 2*Math.PI;
  return a;
}
function hexAlpha(hex, alpha) {
  const r=parseInt(hex.slice(1,3),16), g=parseInt(hex.slice(3,5),16), b=parseInt(hex.slice(5,7),16);
  return `rgba(${r},${g},${b},${alpha})`;
}

// Each object: { x, y, w, h } in pixel coords (top-left corner)
// 0.5 ft = 6 inches = 152.4 mm
const OBJ_MM   = 152.4;           // 0.5 ft in mm
const OBJ_PX   = mmToPx(OBJ_MM); // pixels per object side ≈ 17px
const OBJ_COLORS = ["#fbbf24","#f472b6","#a78bfa","#34d399","#fb923c"];
const NUM_OBJECTS = 5;

// Generate objects with minimum spacing from walls and each other
function generateObjects() {
  const margin  = OBJ_PX + 10;  // clearance from room walls
  const minSep  = OBJ_PX * 2.5; // minimum centre-to-centre separation
  const objs    = [];
  let attempts  = 0;
  while (objs.length < NUM_OBJECTS && attempts < 2000) {
    attempts++;
    const x = WALL.left  + margin + Math.random() * (ROOM_PX - 2*margin - OBJ_PX);
    const y = WALL.top   + margin + Math.random() * (ROOM_PX - 2*margin - OBJ_PX);
    const cx = x + OBJ_PX/2, cy = y + OBJ_PX/2;
    const ok = objs.every(o => {
      const ocx = o.x + OBJ_PX/2, ocy = o.y + OBJ_PX/2;
      return Math.hypot(cx-ocx, cy-ocy) > minSep;
    });
    if (ok) objs.push({ x, y, w: OBJ_PX, h: OBJ_PX });
  }
  return objs;
}

let fieldObjects = generateObjects();

// Test if a point (px,py) is inside any object (with a small padding for robot collision)
function insideAnyObject(px, py, pad=0) {
  return fieldObjects.some(o =>
    px >= o.x - pad && px <= o.x + o.w + pad &&
    py >= o.y - pad && py <= o.y + o.h + pad
  );
}

// rayIntersectRect: returns { enter, exit } t-values for a ray vs AABB, or null if no hit.
// enter = distance to front face, exit = distance to back face.
//
function rayIntersectRect(rx, ry, cos, sin, left, top, right, bottom) {
  // Slab method: find overlap of ray intervals on x and y axes
  let tmin = -Infinity, tmax = Infinity;
  if (Math.abs(cos) > 1e-9) {
    const t1 = (left  - rx) / cos;
    const t2 = (right - rx) / cos;
    tmin = Math.max(tmin, Math.min(t1, t2));
    tmax = Math.min(tmax, Math.max(t1, t2));
  } else {
    if (rx < left || rx > right) return null; // parallel and outside
  }
  if (Math.abs(sin) > 1e-9) {
    const t1 = (top    - ry) / sin;
    const t2 = (bottom - ry) / sin;
    tmin = Math.max(tmin, Math.min(t1, t2));
    tmax = Math.min(tmax, Math.max(t1, t2));
  } else {
    if (ry < top || ry > bottom) return null;
  }
  if (tmax < tmin || tmax < 1e-6) return null; // miss or behind
  return { enter: Math.max(tmin, 1e-6), exit: tmax };
}

// rayToWall: distance to the room wall only (ignores objects — ray passes through them)
function rayToWall(rx, ry, angle) {
  const cos = Math.cos(angle), sin = Math.sin(angle);
  const hit = rayIntersectRect(rx, ry, cos, sin, WALL.left, WALL.top, WALL.right, WALL.bottom);
  // We want the EXIT of the room box (the wall), not the entrance (we're inside)
  return hit ? hit.exit : Infinity;
}

// rayToObjectAndWall: returns { objDist, objIdx, wallDist } all in pixels.
// objDist = distance to nearest object face (null if none intersected within range)
// objIdx  = index into fieldObjects of the object that was hit (-1 if none)
// wallDist = distance to the room wall (always computed, ray passes through objects)
function rayToObjectAndWall(rx, ry, angle) {
  const cos = Math.cos(angle), sin = Math.sin(angle);

  // Wall distance — exit of room AABB
  const wallHit  = rayIntersectRect(rx, ry, cos, sin, WALL.left, WALL.top, WALL.right, WALL.bottom);
  const wallDist = wallHit ? wallHit.exit : Infinity;

  // Object distance — nearest enter face, tracking which object
  let objDist = null, objIdx = -1;
  for (let i = 0; i < fieldObjects.length; i++) {
    const o      = fieldObjects[i];
    const inside = (rx > o.x && rx < o.x+o.w && ry > o.y && ry < o.y+o.h);
    if (inside) continue;
    const hit = rayIntersectRect(rx, ry, cos, sin, o.x, o.y, o.x+o.w, o.y+o.h);
    if (!hit) continue;
    if (hit.enter < wallDist && (objDist === null || hit.enter < objDist)) {
      objDist = hit.enter;
      objIdx  = i;
    }
  }

  return { objDist, objIdx, wallDist };
}

// Each object i has its own particle cloud that localises object i's position

// Each beam hit gives a direct observation of the object centre:
//   face_hit = robot_pos + measuredDist * beamDir
//   centre   = face_hit + (OBJ_PX/2) * beamDir   (step inside by half-width)
//

const N_OBJ_SAMPLES  = 500;   // visual uncertainty cloud sample count
const OBJ_INIT_STD   = 60;   // initial position uncertainty (px) — wide spread
const OBJ_OBS_NOISE  = 2.5;  // observation noise std (px) — how much we trust each hit
const OBJ_DRIFT      = 0.0;  // objects don't move, so process noise = 0

let objEstimates = [];  // { x, y, varX, varY } per object — Kalman state
let objSamples   = [];  // visual cloud: array of {x,y} per object
let objHistory   = [];
let objSeenCount = [];  // how many observations fused per object

function initObjParticles() {
  // Start with wide uniform uncertainty centred on room centre
  const cx = WALL.left + ROOM_PX / 2;
  const cy = WALL.top  + ROOM_PX / 2;
  objEstimates = fieldObjects.map(() => ({
    x: cx, y: cy,
    varX: OBJ_INIT_STD * OBJ_INIT_STD,
    varY: OBJ_INIT_STD * OBJ_INIT_STD,
  }));
  objSamples   = fieldObjects.map(() => _sampleCloud(cx, cy, OBJ_INIT_STD, OBJ_INIT_STD));
  objHistory   = fieldObjects.map(() => []);
  objSeenCount = fieldObjects.map(() => 0);
}

function _sampleCloud(mx, my, stdX, stdY) {
  const out = [];
  for (let i = 0; i < N_OBJ_SAMPLES; i++) {
    out.push({
      x: clamp(gauss(mx, stdX), WALL.left, WALL.right),
      y: clamp(gauss(my, stdY), WALL.top,  WALL.bottom),
    });
  }
  return out;
}

function updateObjParticles() {
  // For each beam that hit an object, compute the direct geometric observation
  // of that object's centre and fuse it into the object's Kalman estimate.
  const obsNoiseSq = OBJ_OBS_NOISE * OBJ_OBS_NOISE;

  BEAM_OFFSETS.forEach((offset, b) => {
    const idx = beamObjIdx[b];
    if (idx < 0 || beamReadings[b].obj === null) return; // no hit this beam

    const beamDir  = robot.theta + offset;
    const distPx   = mmToPx(inToMm(beamReadings[b].obj));  // measured dist in px

    // The beam hit the FACE of the object. Step inward by half the object width
    // to get the estimated object CENTRE.
    const facX = robot.x + distPx * Math.cos(beamDir);
    const facY = robot.y + distPx * Math.sin(beamDir);
    const obsX = facX + (OBJ_PX / 2) * Math.cos(beamDir);
    const obsY = facY + (OBJ_PX / 2) * Math.sin(beamDir);

    // 1-D Kalman update independently on x and y
    const est = objEstimates[idx];

    // Kalman gain  K = var / (var + R)
    const Kx = est.varX / (est.varX + obsNoiseSq);
    const Ky = est.varY / (est.varY + obsNoiseSq);

    est.x    += Kx * (obsX - est.x);
    est.y    += Ky * (obsY - est.y);
    est.varX  = (1 - Kx) * est.varX;
    est.varY  = (1 - Ky) * est.varY;

    objSeenCount[idx]++;
  });

  // Rebuild visual uncertainty clouds from current estimates
  objSamples = objEstimates.map(est => {
    const stdX = Math.sqrt(Math.max(est.varX, 0.1));
    const stdY = Math.sqrt(Math.max(est.varY, 0.1));
    return _sampleCloud(est.x, est.y, stdX, stdY);
  });

  // Track accuracy history
  fieldObjects.forEach((o, i) => {
    const err = pxToIn(Math.hypot(
      objEstimates[i].x - (o.x + o.w / 2),
      objEstimates[i].y - (o.y + o.h / 2)
    ));
    objHistory[i] = [...(objHistory[i] || []).slice(-40), err];
  });
}

// Accuracy: distance from each object's estimate to the true object centre.
function computeObjAccuracy() {
  return fieldObjects.map((o, i) => {
    const ocx = o.x + o.w / 2, ocy = o.y + o.h / 2;
    const est = objEstimates[i];
    return { errorIn: pxToIn(Math.hypot(est.x - ocx, est.y - ocy)) };
  });
}
let particles  = [];
let robot      = { x: WALL.left + ROOM_PX/2, y: WALL.top + ROOM_PX/2, theta: 0 };
let estimate   = { x: robot.x, y: robot.y };
let stepCount  = 0;
let errorVal   = 0;
let history    = [];

let robotHistory = [];  // best error seen so far, running avg, convergence step
let beamReadings = [{obj:null,wall:null},{obj:null,wall:null},{obj:null,wall:null},{obj:null,wall:null}];

// Scatter particles uniformly in the room.
// With a gyro, we initialise all particles to the robot's known heading + tiny drift
// This eliminates rotational ambiguity from the very start
function initParticles() {
  particles = [];
  let attempts = 0;
  while (particles.length < NUM_PARTICLES && attempts < NUM_PARTICLES * 10) {
    attempts++;
    const x = WALL.left + Math.random() * ROOM_PX;
    const y = WALL.top  + Math.random() * ROOM_PX;
    if (insideAnyObject(x, y, 2)) continue; // skip positions inside objects
    particles.push({
      x, y,
      theta: robot.theta + gauss(0, GYRO_NOISE),
      w:     1 / NUM_PARTICLES,
    });
  }
  // Fill any remaining slots if we ran out of attempts
  while (particles.length < NUM_PARTICLES) {
    particles.push({
      x: WALL.left + Math.random() * ROOM_PX,
      y: WALL.top  + Math.random() * ROOM_PX,
      theta: robot.theta + gauss(0, GYRO_NOISE),
      w: 1 / NUM_PARTICLES,
    });
  }
}

// Each beam returns { obj, wall }:
//   obj  = distance to nearest object face in inches (null if none in range)
//   wall = distance to room wall in inches (null if out of sensor range)
// Also populates beamObjIdx[b] with the index of whichever object was hit (-1 if none).
function senseBeams(rx, ry, theta) {
  const rangePx = sensorRangePx();
  beamObjIdx = [-1, -1, -1, -1]; // reset attribution
  return BEAM_OFFSETS.map((offset, b) => {
    const beamDir = theta + offset;
    const { objDist, objIdx, wallDist } = rayToObjectAndWall(rx, ry, beamDir);

    const wallIn = wallDist <= rangePx
      ? pxToIn(wallDist) + gauss(0, SENSOR_NOISE_IN)
      : null;

    const objIn = (objDist !== null && objDist <= rangePx)
      ? pxToIn(objDist) + gauss(0, SENSOR_NOISE_IN)
      : null;

    if (objIn !== null) beamObjIdx[b] = objIdx; // record which object this beam hit

    return { obj: objIn, wall: wallIn };
  });
}

function moveRobot(fwdPx, turn) {
  robot.theta += turn;
  const nx = clamp(robot.x + fwdPx * Math.cos(robot.theta), WALL.left + 8,  WALL.right  - 8);
  const ny = clamp(robot.y + fwdPx * Math.sin(robot.theta), WALL.top  + 8,  WALL.bottom - 8);
  // Only move if not entering an object (robot radius ~8px)
  if (!insideAnyObject(nx, ny, 8)) {
    robot.x = nx; robot.y = ny;
  }
}

// With a gyro, each particle's heading is set to the robot's measured heading
// plus a small gyro drift noise 
// This keeps all particles rotationally aligned, eliminating the biggest source
// of divergence in a symmetric room.
function moveParticles(fwdPx, turn) {
  particles = particles.map(p => {
    // Gyro gives the true heading — particle theta tracks it with tiny drift
    const theta = robot.theta + gauss(0, GYRO_NOISE);
    return {
      ...p, theta,
      x: clamp(p.x + fwdPx*Math.cos(theta) + gauss(0, MOTION_NOISE*0.5), WALL.left, WALL.right),
      y: clamp(p.y + fwdPx*Math.sin(theta) + gauss(0, MOTION_NOISE*0.5), WALL.top,  WALL.bottom),
    };
  });
}

// Each beam provides up to 2 measurements: object distance and wall distance.
// Each particle predicts both using its own ray cast, scored via Gaussian log-likelihood.
// More valid measurements = stronger weight signal = faster convergence.
function updateWeights(measurement) {
  let total    = 0;
  // Count beams where at least one reading (obj or wall) is valid
  let seenCount = measurement.filter(m => m.wall !== null || m.obj !== null).length;

  const sigSq = SENSOR_NOISE_IN * SENSOR_NOISE_IN;

  particles = particles.map(p => {
    let logW     = 0;
    let compared = 0;

    BEAM_OFFSETS.forEach((offset, b) => {
      const m = measurement[b];
      const beamDir = p.theta + offset;
      const { objDist: pObjPx, wallDist: pWallPx } = rayToObjectAndWall(p.x, p.y, beamDir);

      // Score wall measurement
      if (m.wall !== null) {
        const predWallIn = pxToIn(pWallPx);
        logW += -((predWallIn - m.wall) ** 2) / (2 * sigSq);
        compared++;
      }

      // Score object measurement
      if (m.obj !== null) {
        // particle predicts an object on this beam?
        const predObjIn = pObjPx !== null ? pxToIn(pObjPx) : null;
        if (predObjIn !== null) {
          // Both saw an object — score the distance match
          logW += -((predObjIn - m.obj) ** 2) / (2 * sigSq);
        } else {
          // Robot saw an object but particle didn't — penalise
          logW += -4.0;
        }
        compared++;
      } else if (m.wall !== null) {
        // Robot saw no object; penalise if particle predicts one (phantom object)
        if (pObjPx !== null && pxToIn(pObjPx) <= SENSOR_RANGE_IN) {
          logW += -2.0;
        }
      }
    });

    const w = compared > 0 ? Math.exp(logW) : 0.0001;
    total  += w;
    return { ...p, w };
  });

  particles = particles.map(p => ({ ...p, w: p.w / (total || 1) }));
  return seenCount;
}

function resample() {
  const next = [];
  for (let i = 0; i < NUM_PARTICLES; i++) {
    const r = Math.random();
    let cum = 0, chosen = particles[particles.length-1];
    for (const p of particles) {
      cum += p.w;
      if (r <= cum) { chosen = p; break; }
    }
    next.push({
      ...chosen,
      x:     clamp(chosen.x + gauss(0, RESAMPLE_NOISE), WALL.left, WALL.right),
      y:     clamp(chosen.y + gauss(0, RESAMPLE_NOISE), WALL.top,  WALL.bottom),
      theta: chosen.theta + gauss(0, 0.01),
      w:     1 / NUM_PARTICLES,
    });
  }
  particles = next;
}

function estimatePos() {
  let x=0, y=0;
  particles.forEach(p => { x+=p.x; y+=p.y; });
  return { x: x/NUM_PARTICLES, y: y/NUM_PARTICLES };
}

function particleSpread() {
  let mx=0, my=0;
  particles.forEach(p => { mx+=p.x; my+=p.y; });
  mx/=NUM_PARTICLES; my/=NUM_PARTICLES;
  let v=0;
  particles.forEach(p => { v+=(p.x-mx)**2+(p.y-my)**2; });
  return Math.sqrt(v/NUM_PARTICLES);
}

function runStep(fwdPx, turn) {
  moveRobot(fwdPx, turn);
  beamReadings = senseBeams(robot.x, robot.y, robot.theta);
  moveParticles(fwdPx, turn);
  const seen = updateWeights(beamReadings);
  resample();
  estimate  = estimatePos();
  errorVal  = pxToIn(dist(robot.x, robot.y, estimate.x, estimate.y));
  stepCount++;
  history      = [...history.slice(-40), errorVal];
  robotHistory = [...robotHistory, errorVal];

  updateObjParticles();

  updateUI(seen);
  draw();
  updateObjAccuracyUI();

  document.getElementById("beam-warn").classList.toggle("show", seen === 0);
}

function updateObjAccuracyUI() {
  const acc = computeObjAccuracy();
  const container = document.getElementById("obj-accuracy");
  container.innerHTML = acc.map((a, i) => {
    const color = OBJ_COLORS[i];
    const err   = a.errorIn.toFixed(2) + " in";
    const good  = a.errorIn < 6;
    return `
      <div style="display:flex;justify-content:space-between;align-items:center;font-size:11px;margin-bottom:3px;">
        <span style="display:flex;align-items:center;gap:5px;color:#94a3b8;">
          <span style="width:8px;height:8px;background:${color};display:inline-block;border-radius:1px;"></span>
          Object ${i+1}
        </span>
        <span style="font-weight:700;color:${good ? '#34d399' : '#fb923c'}">${err}</span>
      </div>`;
  }).join("");
}

//ui
function updateUI(seen) {
  document.getElementById("s-step").textContent    = stepCount;
  document.getElementById("s-error").textContent   = errorVal.toFixed(2) + " in";
  document.getElementById("s-fov").textContent     = BEAM_WIDTH_DEG + "°";
  document.getElementById("s-range").textContent   = SENSOR_RANGE_IN + " in";
  document.getElementById("s-rx").textContent      = pxToIn(robot.x - WALL.left).toFixed(1) + " in";
  document.getElementById("s-ry").textContent      = pxToIn(robot.y - WALL.top).toFixed(1)  + " in";
  document.getElementById("s-heading").textContent = (robot.theta * 180 / Math.PI).toFixed(1) + "°";
  const beamsEl = document.getElementById("s-beams");
  beamsEl.textContent = `${seen} / 4`;
  beamsEl.className   = "stat-val" + (seen === 0 ? " warn" : "");

  BEAM_IDS.forEach((id, b) => {
    const el = document.getElementById(id);
    const r  = beamReadings[b];
    if (!r || (r.wall === null && r.obj === null)) {
      el.textContent = "—";
    } else {
      const wallStr = r.wall !== null ? r.wall.toFixed(1) + "in" : "—";
      const objStr  = r.obj  !== null ? " | obj:" + r.obj.toFixed(1) + "in" : "";
      el.textContent = wallStr + objStr;
    }
  });
//accuracy
  if (robotHistory.length > 0) {
    const best    = Math.min(...robotHistory);
    const recent  = robotHistory.slice(-20);
    const avg     = recent.reduce((s,v) => s+v, 0) / recent.length;
    // "Converged" = first step where error dropped below 6in and stayed ≤10in for 5 steps
    let convStep  = "—";
    for (let k = 4; k < robotHistory.length; k++) {
      if (robotHistory.slice(k-4, k+1).every(e => e <= 6)) { convStep = k - 3; break; }
    }
    const curGood  = errorVal <= 6;
    const bestGood = best     <= 6;
    document.getElementById("ra-cur").textContent  = errorVal.toFixed(2) + " in";
    document.getElementById("ra-cur").className    = "stat-val" + (curGood  ? "" : " warn");
    document.getElementById("ra-best").textContent = best.toFixed(2) + " in";
    document.getElementById("ra-best").className   = "stat-val" + (bestGood ? "" : " warn");
    document.getElementById("ra-avg").textContent  = avg.toFixed(2) + " in";
    document.getElementById("ra-conv").textContent = convStep === "—" ? "—" : "step " + convStep;
  }

  drawChart();
}

function drawChart() {
  const svg = document.getElementById("chart");
  if (history.length < 2) return;
  const maxE = Math.max(...history, 1);
  const pts  = history.map((e,i) => `${(i/(history.length-1))*182},${55-(e/maxE)*50}`).join(" ");
  const last = history[history.length-1];
  svg.innerHTML = `
    <polyline points="${pts}" fill="none" stroke="#38bdf8" stroke-width="1.5" stroke-linejoin="round"/>
    <circle cx="182" cy="${55-(last/maxE)*50}" r="3" fill="#38bdf8"/>`;
}

const canvas = document.getElementById("canvas");
const ctx    = canvas.getContext("2d");

function draw() {
  ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

  // Dark background
  ctx.fillStyle = "#0a0e1a";
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

  ctx.strokeStyle = "#475569";
  ctx.lineWidth   = 3;
  ctx.strokeRect(WALL.left, WALL.top, ROOM_PX, ROOM_PX);

  // Wall fill (very subtle)
  ctx.fillStyle = "rgba(71,85,105,0.06)";
  ctx.fillRect(WALL.left, WALL.top, ROOM_PX, ROOM_PX);

  // Wall dimension labels
  ctx.font      = "10px 'Courier New'";
  ctx.fillStyle = "#475569";
  ctx.textAlign = "center";
  ctx.fillText("12 ft / 3600 mm", WALL.left + ROOM_PX/2, WALL.top - 6);
  ctx.save();
  ctx.translate(WALL.left - 6, WALL.top + ROOM_PX/2);
  ctx.rotate(-Math.PI/2);
  ctx.fillText("12 ft / 3600 mm", 0, 0);
  ctx.restore();
  ctx.textAlign = "left";

  // Grid lines inside room (every 300mm = 1ft ≈ 33px)
  const gridStepPx = mmToPx(300);
  ctx.strokeStyle = "rgba(255,255,255,0.04)";
  ctx.lineWidth   = 1;
  for (let i = 1; i < 12; i++) {
    const x = WALL.left + i * gridStepPx;
    const y = WALL.top  + i * gridStepPx;
    ctx.beginPath(); ctx.moveTo(x, WALL.top);    ctx.lineTo(x, WALL.bottom); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(WALL.left, y);   ctx.lineTo(WALL.right, y);  ctx.stroke();
  }

  fieldObjects.forEach((o, i) => {
    const color = OBJ_COLORS[i];
    // Fill
    ctx.fillStyle = hexAlpha(color, 0.12);
    ctx.fillRect(o.x, o.y, o.w, o.h);
    // Border
    ctx.strokeStyle = hexAlpha(color, 0.7);
    ctx.lineWidth   = 1.5;
    ctx.strokeRect(o.x, o.y, o.w, o.h);
    // Label
    ctx.font      = "bold 9px 'Courier New'";
    ctx.fillStyle = hexAlpha(color, 0.9);
    ctx.textAlign = "center";
    ctx.fillText(`O${i+1}`, o.x + o.w/2, o.y + o.h/2 + 3);
    ctx.textAlign = "left";
    // Size annotation on first object only to avoid clutter
    if (i === 0) {
      ctx.font      = "8px 'Courier New'";
      ctx.fillStyle = hexAlpha(color, 0.5);
      ctx.textAlign = "center";
      ctx.fillText("0.5ft", o.x + o.w/2, o.y - 3);
      ctx.textAlign = "left";
    }
  });

  particles.forEach(p => {
    const alpha = Math.min(1, p.w * NUM_PARTICLES * 2);
    ctx.beginPath();
    ctx.arc(p.x, p.y, 3.5, 0, Math.PI * 2);
    ctx.fillStyle = `rgba(200,220,255,${0.25 + alpha * 0.75})`;
    ctx.fill();
  });

  // objSamples[i] = Gaussian cloud drawn around the Kalman estimate for object i.
  // Tight cloud = confident; wide cloud = uncertain (few observations so far).
  objSamples.forEach((samples, i) => {
    const color = OBJ_COLORS[i];
    samples.forEach(s => {
      ctx.beginPath();
      ctx.arc(s.x, s.y, 3, 0, Math.PI * 2);
      ctx.fillStyle = hexAlpha(color, 0.55);
      ctx.fill();
    });

    // Estimated object position — ring + dot
    const est = objEstimates[i];
    ctx.beginPath();
    ctx.arc(est.x, est.y, 7, 0, Math.PI * 2);
    ctx.strokeStyle = hexAlpha(color, 0.95);
    ctx.lineWidth   = 2;
    ctx.stroke();
    ctx.beginPath();
    ctx.arc(est.x, est.y, 2.5, 0, Math.PI * 2);
    ctx.fillStyle = hexAlpha(color, 0.95);
    ctx.fill();

    // Observation count badge
    if (objSeenCount[i] > 0) {
      ctx.font      = "8px 'Courier New'";
      ctx.fillStyle = hexAlpha(color, 0.7);
      ctx.textAlign = "center";
      ctx.fillText("n=" + objSeenCount[i], est.x, est.y - 11);
      ctx.textAlign = "left";
    }
  });

  // Each beam now passes THROUGH objects to the wall.
  // The cone extends to the wall. Object hits are shown as separate diamonds on the ray.
  const halfAngle   = BEAM_WIDTH_DEG * Math.PI / 180;
  const rangePx     = sensorRangePx();

  BEAM_OFFSETS.forEach((offset, b) => {
    const beamDir = robot.theta + offset;
    const { objDist: objPx, wallDist: wallPx } = rayToObjectAndWall(robot.x, robot.y, beamDir);
    const inRange   = wallPx <= rangePx;
    const drawDist  = Math.min(wallPx, rangePx); // full length to wall (clipped to range)
    const color     = BEAM_COLORS[b];
    const reading   = beamReadings[b];

    // Beam cone — full width to wall
    ctx.beginPath();
    ctx.moveTo(robot.x, robot.y);
    ctx.arc(robot.x, robot.y, drawDist, beamDir - halfAngle, beamDir + halfAngle);
    ctx.closePath();
    ctx.fillStyle   = hexAlpha(color, inRange ? 0.06 : 0.03);
    ctx.strokeStyle = hexAlpha(color, inRange ? 0.3  : 0.10);
    ctx.lineWidth   = 1;
    ctx.fill();
    ctx.stroke();

    if (inRange) {
      const wallHitX = robot.x + wallPx * Math.cos(beamDir);
      const wallHitY = robot.y + wallPx * Math.sin(beamDir);

      // Solid ray all the way to the wall
      ctx.beginPath();
      ctx.moveTo(robot.x, robot.y);
      ctx.lineTo(wallHitX, wallHitY);
      ctx.strokeStyle = hexAlpha(color, 0.55);
      ctx.lineWidth   = 1.5;
      ctx.setLineDash([]);
      ctx.stroke();

      // Wall hit dot
      ctx.beginPath();
      ctx.arc(wallHitX, wallHitY, 4, 0, Math.PI * 2);
      ctx.fillStyle = hexAlpha(color, 0.9);
      ctx.fill();

      // Wall distance label
      if (reading && reading.wall !== null) {
        ctx.font      = "9px 'Courier New'";
        ctx.fillStyle = hexAlpha(color, 0.75);
        ctx.textAlign = "center";
        const lx = robot.x + (wallPx * 0.65) * Math.cos(beamDir);
        const ly = robot.y + (wallPx * 0.65) * Math.sin(beamDir);
        ctx.fillText(reading.wall.toFixed(1) + "in", lx, ly - 5);
        ctx.textAlign = "left";
      }

      // Object hit — diamond marker on the ray + separate label
      if (objPx !== null && objPx <= rangePx) {
        const ox = robot.x + objPx * Math.cos(beamDir);
        const oy = robot.y + objPx * Math.sin(beamDir);
        const ds = 5; // diamond half-size
        ctx.beginPath();
        ctx.moveTo(ox, oy - ds);
        ctx.lineTo(ox + ds, oy);
        ctx.lineTo(ox, oy + ds);
        ctx.lineTo(ox - ds, oy);
        ctx.closePath();
        ctx.fillStyle   = "#ffffff";
        ctx.strokeStyle = hexAlpha(color, 1.0);
        ctx.lineWidth   = 1.5;
        ctx.fill();
        ctx.stroke();

        // Dashed segment from robot to object (shows obj portion of beam)
        ctx.beginPath();
        ctx.moveTo(robot.x, robot.y);
        ctx.lineTo(ox, oy);
        ctx.strokeStyle = hexAlpha(color, 0.9);
        ctx.lineWidth   = 2;
        ctx.setLineDash([3, 3]);
        ctx.stroke();
        ctx.setLineDash([]);

        if (reading && reading.obj !== null) {
          ctx.font      = "9px 'Courier New'";
          ctx.fillStyle = "#ffffff";
          ctx.textAlign = "center";
          const lx2 = robot.x + (objPx * 0.5) * Math.cos(beamDir);
          const ly2 = robot.y + (objPx * 0.5) * Math.sin(beamDir);
          ctx.fillText("◆" + reading.obj.toFixed(1) + "in", lx2, ly2 - 6);
          ctx.textAlign = "left";
        }
      }
    } else {
      // Dashed ray stopping at range limit
      ctx.beginPath();
      ctx.moveTo(robot.x, robot.y);
      ctx.lineTo(robot.x + rangePx*Math.cos(beamDir), robot.y + rangePx*Math.sin(beamDir));
      ctx.strokeStyle = hexAlpha(color, 0.2);
      ctx.lineWidth   = 1;
      ctx.setLineDash([4, 6]);
      ctx.stroke();
      ctx.setLineDash([]);
    }

    // Beam label (F/R/B/L) at tip of range arc
    const labelR = Math.min(drawDist, rangePx) + 12;
    ctx.font      = "bold 10px 'Courier New'";
    ctx.fillStyle = hexAlpha(color, inRange ? 0.8 : 0.3);
    ctx.textAlign = "center";
    ctx.fillText(BEAM_LABELS[b],
      robot.x + labelR * Math.cos(beamDir),
      robot.y + labelR * Math.sin(beamDir) + 4);
    ctx.textAlign = "left";
  });

  ctx.setLineDash([]);

  // ── Estimated position ─────────────────────────────────────────────────────
  ctx.beginPath();
  ctx.arc(estimate.x, estimate.y, 10, 0, Math.PI * 2);
  ctx.strokeStyle = "rgba(52,211,153,0.7)"; ctx.lineWidth = 2; ctx.stroke();
  ctx.beginPath();
  ctx.arc(estimate.x, estimate.y, 4, 0, Math.PI * 2);
  ctx.fillStyle = "#34d399"; ctx.fill();

  // ── True robot ─────────────────────────────────────────────────────────────
  ctx.save();
  ctx.translate(robot.x, robot.y);
  ctx.rotate(robot.theta);
  ctx.beginPath();
  ctx.moveTo(12,0); ctx.lineTo(-8,7); ctx.lineTo(-8,-7); ctx.closePath();
  ctx.fillStyle = "#f87171"; ctx.strokeStyle = "#fca5a5"; ctx.lineWidth = 1.5;
  ctx.fill(); ctx.stroke();
  ctx.restore();
}

// ─── WASD input ───────────────────────────────────────────────────────────────
const keys = {};
document.addEventListener("keydown", e => {
  const k = e.key.toLowerCase();
  if (["w","a","s","d"].includes(k)) { e.preventDefault(); keys[k] = true; }
});
document.addEventListener("keyup", e => {
  keys[e.key.toLowerCase()] = false;
});

let lastTime = 0;
function gameLoop(ts) {
  requestAnimationFrame(gameLoop);
  if (ts - lastTime < 80) return;
  lastTime = ts;
  const fwdPx = ((keys["w"] ? 1 : 0) - (keys["s"] ? 1 : 0)) * mmToPx(100);
  const turn  = (keys["d"] ? 0.13 : 0) - (keys["a"] ? 0.13 : 0);
  if (fwdPx !== 0 || turn !== 0) runStep(fwdPx, turn);
}
requestAnimationFrame(gameLoop);

// ─── Reset ────────────────────────────────────────────────────────────────────
document.getElementById("btn-reset").addEventListener("click", () => {
  fieldObjects = generateObjects();
  // Pick a robot start position not inside any object
  let rx = WALL.left + ROOM_PX/2, ry = WALL.top + ROOM_PX/2;
  for (let t = 0; t < 200 && insideAnyObject(rx, ry, 10); t++) {
    rx = WALL.left + 30 + Math.random() * (ROOM_PX - 60);
    ry = WALL.top  + 30 + Math.random() * (ROOM_PX - 60);
  }
  robot    = { x: rx, y: ry, theta: 0 };
  estimate = { x: robot.x, y: robot.y };
  stepCount=0; errorVal=0; history=[]; robotHistory=[]; beamReadings=[{obj:null,wall:null},{obj:null,wall:null},{obj:null,wall:null},{obj:null,wall:null}];
  initParticles(); initObjParticles(); updateUI(0); updateObjAccuracyUI(); draw();
});

// ─── Sliders ──────────────────────────────────────────────────────────────────
document.getElementById("sl-fov").addEventListener("input", e => {
  BEAM_WIDTH_DEG = +e.target.value;
  document.getElementById("lbl-fov").textContent = BEAM_WIDTH_DEG + "°";
  document.getElementById("s-fov").textContent   = BEAM_WIDTH_DEG + "°";
  draw();
});
document.getElementById("sl-range").addEventListener("input", e => {
  SENSOR_RANGE_IN = +e.target.value;
  document.getElementById("lbl-range").textContent = SENSOR_RANGE_IN;
  document.getElementById("s-range").textContent   = SENSOR_RANGE_IN + " in";
  draw();
});
document.getElementById("sl-noise").addEventListener("input", e => {
  SENSOR_NOISE_IN = +e.target.value;
  document.getElementById("lbl-noise").textContent = (+e.target.value).toFixed(1);
});

document.getElementById("sl-gyro").addEventListener("input", e => {
  GYRO_NOISE = +e.target.value;
  document.getElementById("lbl-gyro").textContent = (+e.target.value).toFixed(3);
});

// ─── Boot ─────────────────────────────────────────────────────────────────────
// Make sure initial robot position isn't inside an object
(function () {
  let rx = robot.x, ry = robot.y;
  for (let t = 0; t < 200 && insideAnyObject(rx, ry, 10); t++) {
    rx = WALL.left + 30 + Math.random() * (ROOM_PX - 60);
    ry = WALL.top  + 30 + Math.random() * (ROOM_PX - 60);
  }
  robot.x = rx; robot.y = ry;
  estimate.x = rx; estimate.y = ry;
})();
initParticles();
initObjParticles();
draw();
updateUI(0, 0);
updateObjAccuracyUI();
</script>
</body>
</html>
