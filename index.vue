<template>
  <div class="a" @touchstart="ts" @touchmove="tm" @touchend="te">
    <header class="b">
      <h1>VoxExplore 3D</h1>
      <p>Pos: <b>{{ Math.floor(px) }}, {{ Math.floor(py) }}</b> | Alt: <b>{{ pz > 0 ? 'Airborne' : 'Ground' }}</b></p>
    </header>

    <main class="c">
      <canvas ref="g" width="320" height="240"></canvas>
      <div class="cross"></div>
    </main>

    <footer class="f">
      <div class="d">
        <!-- Giant Tactile Mobile Jump Action Button Matrix -->
        <button @touchstart.prevent="jp" class="jmp-btn">JUMP</button>
        <div class="pad-spacer"></div>
        <!-- Mobile Cross D-Pad Shell Layout (L/R Turns, U/D Moves) -->
        <div class="dpad" ref="dpBox">
          <button class="p-btn u" @touchstart.prevent="vPad('u',1)" @touchend.prevent="vPad('u',0)" @touchcancel.prevent="vPad('u',0)">▲</button>
          <button class="p-btn l" @touchstart.prevent="vPad('l',1)" @touchend.prevent="vPad('l',0)" @touchcancel.prevent="vPad('l',0)">◀</button>
          <div class="center-cap"></div>
          <button class="p-btn r" @touchstart.prevent="vPad('r',1)" @touchend.prevent="vPad('r',0)" @touchcancel.prevent="vPad('r',0)">▶</button>
          <button class="p-btn dwn" @touchstart.prevent="vPad('d',1)" @touchend.prevent="vPad('d',0)" @touchcancel.prevent="vPad('d',0)">▼</button>
        </div>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const g = ref(null), dpBox = ref(null)
const mClrs = { '1': '#8b5a2b', '2': '#708090', '3': '#a0522d', '4': '#228b22' }

let ctx, tLoop, px = 2.5, py = 2.5, pz = 0, pVz = 0, pa = 0, sx = 0, sy = 0, map = []
let dyVector = 0, dxVector = 0
const mapW = 16, mapH = 16

// Procedural Voxel Landscape Generator Simulation
const initWorld = () => {
  for (let i = 0; i < mapW * mapH; i++) {
    let x = i % mapW, y = Math.floor(i / mapW)
    if (x === 0 || x === mapW - 1 || y === 0 || y === mapH - 1) map[i] = '2' // Stone borders
    else if ((x + y) % 5 === 0) map[i] = '3' // Wood ridges
    else if ((x * y) % 7 === 3) map[i] = '4' // Leafy formations
    else if (x > 6 && x < 10 && y > 6 && y < 10) map[i] = '1' // Central dirt plateau hills
    else map[i] = '0'
  }
}

// Fixed D-Pad Trigger Configurations (L/R Turn Angle, U/D Move Positions)
const vPad = (dir, val) => {
  if (dir === 'u') dyVector = val * 0.075
  if (dir === 'd') dyVector = val * -0.075
  if (dir === 'l') dxVector = val * -0.045
  if (dir === 'r') dxVector = val * 0.045
}

// Direct Gravity Physics Jump Impulse Pipeline
const jp = () => {
  if (pz <= 0) { pVz = 0.28 } // Vertical jump force impulse factor
}

// Mobile 3D Scene Swiping Engine: Drag X turns, Drag Y moves player
const ts = (e) => { 
  if (!e.touches.length) return
  const t = e.touches
  if (!dpBox.value || t.clientX < dpBox.value.getBoundingClientRect().left - 20) {
    sx = t.clientX; sy = t.clientY 
  }
}
const tm = (e) => {
  if ((!sx && !sy) || !e.touches.length) return
  const t = e.touches
  let dx = t.clientX - sx, dy = t.clientY - sy
  
  pa += dx * 0.009 // Horizontal turns view angle
  
  let walkSpeed = dy * -0.018 // Vertical walks forward/backward
  let nx = px + Math.cos(pa) * walkSpeed
  let ny = py + Math.sin(pa) * walkSpeed
  
  if (checkMove(nx, ny)) { px = nx; py = ny }
  sx = t.clientX; sy = t.clientY
}
const te = () => { sx = 0; sy = 0 }

// Physics Engine Voxel Collision Matrix Checking
const checkMove = (nx, ny) => {
  let mx = Math.floor(nx), my = Math.floor(ny)
  if (mx < 0 || mx >= mapW || my < 0 || my >= mapH) return false
  let cell = map[my * mapW + mx]
  // Can only traverse block coordinates if completely empty or jumping higher than the block plane height
  return cell === '0' || pz > 0.45
}

const tick = () => {
  // Update rotation viewpoint turning angle
  if (dxVector !== 0) pa += dxVector

  // Step position update engine loops
  if (dyVector !== 0) {
    let nx = px + Math.cos(pa) * dyVector
    let ny = py + Math.sin(pa) * dyVector
    if (checkMove(nx, ny)) { px = nx; py = ny }
  }

  // Real-time Gravity Physics Solver Engine
  pz += pVz
  if (pz > 0) {
    pVz -= 0.025 // Constant falling acceleration factor (Gravity)
    // Check if player landed on a solid voxel block mid-air
    let curCell = map[Math.floor(py) * mapW + Math.floor(px)]
    if (curCell !== '0' && pz <= 0.5) {
      pz = 0.5 // Remain snapped standing on top of voxel block height
      pVz = 0
    }
  } else {
    // Ground level floor bounds snapping constraint limits
    pz = 0; pVz = 0
    let curCell = map[Math.floor(py) * mapW + Math.floor(px)]
    if (curCell !== '0') pz = 0.5 // Safe-snap upward to solid terrain if clipping inside
  }

  // Clear Canvas Viewport & Render Horizon Shading
  ctx.fillStyle = '#bae6fd'; ctx.fillRect(0, 0, 320, 120)
  ctx.fillStyle = '#1c1917'; ctx.fillRect(0, 120, 320, 120)

  // Height-Aware 3D Raycasting Slice Render Projection Framework
  for (let x = 0; x < 320; x++) {
    let rAngle = (pa - 0.5) + (x / 320), rx = px, ry = py
    let sin = Math.sin(rAngle), cos = Math.cos(rAngle), d = 0, hit = '0'

    while (d < 16) {
      rx += cos * 0.05; ry += sin * 0.05; d += 0.05
      let cell = map[Math.floor(ry) * mapW + Math.floor(rx)]
      if (cell && cell !== '0') { hit = cell; break }
    }

    if (hit !== '0') {
      let correctedDist = d * Math.cos(rAngle - pa)
      // Visual spatial vertical offset projection tracking player vertical jump jump height scaling
      let vOffset = Math.floor((pz * 90) / correctedDist)
      let wallH = Math.min(240, Math.floor(180 / correctedDist))
      
      let y0 = Math.max(0, (120 - wallH / 2) + vOffset)
      
      ctx.fillStyle = mClrs[hit] || '#708090'
      ctx.globalAlpha = Math.max(0.2, 1 - (d / 12))
      ctx.fillRect(x, y0, 1, wallH)
      ctx.globalAlpha = 1.0
    }
  }
}

onMounted(() => {
  ctx = g.value.getContext('2d')
  initWorld()
  tLoop = setInterval(tick, 1000 / 60)

  window.addEventListener('keydown', e => {
    if (e.key === 'w' || e.key === 'ArrowUp') dyVector = 0.075
    if (e.key === 's' || e.key === 'ArrowDown') dyVector = -0.075
    if (e.key === 'a' || e.key === 'ArrowLeft') dxVector = -0.045
    if (e.key === 'd' || e.key === 'ArrowRight') dxVector = 0.045
    if (e.key === ' ') jp()
  })
  window.addEventListener('keyup', e => {
    if (['w', 's', 'ArrowUp', 'ArrowDown'].includes(e.key)) dyVector = 0
    if (['a', 'd', 'ArrowLeft', 'ArrowRight'].includes(e.key)) dxVector = 0
  })
})
onUnmounted(() => clearInterval(tLoop))
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 12px; box-sizing: border-box; font-family: sans-serif; user-select: none; overflow: hidden; }
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 18px; text-transform: uppercase; color: #e2e8f0; } p { margin: 2px 0 0 0; color: #71717a; font-size: 11px; } b { color: #38bdf8; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #27272a; background: #111; overflow: hidden; }
canvas { display: block; width: 100%; height: 100%; image-rendering: pixelated; }
.cross { position: absolute; top: 50%; left: 50%; width: 4px; height: 4px; background: #fff; transform: translate(-50%, -50%); pointer-events: none; opacity: 0.4; }

.f { width: 100%; max-width: 320px; padding-top: 10px; }
.d { display: flex; align-items: center; width: 100%; height: 130px; }
.pad-spacer { flex: 1; }

/* Large Fluid Tactical Ergonomic Jump Controller Button */
.jmp-btn { width: 76px; height: 76px; background: #27272a; border: 3px solid #3f3f46; color: #f4f4f5; font-weight: 800; font-size: 13px; border-radius: 50%; cursor: pointer; touch-action: none; box-shadow: 0 6px 12px rgba(0,0,0,0.6); letter-spacing: 0.5px; }
.jmp-btn:active { background: #38bdf8; color: #000; border-color: #bae6fd; transform: scale(0.95); }

/* Oversized Mobile D-Pad Cross Shell */
.dpad { position: relative; width: 124px; height: 124px; display: flex; align-items: center; justify-content: center; }
.center-cap { width: 42px; height: 42px; background: #18181b; border: 1px solid #27272a; z-index: 2; pointer-events: none; }
.p-btn { position: absolute; background: #27272a; border: 2px solid #3f3f46; color: #f4f4f5; display: flex; align-items: center; justify-content: center; font-size: 16px; font-weight: 900; touch-action: none; z-index: 3; box-shadow: 0 4px 6px rgba(0,0,0,0.4); }
.p-btn:active { background: #38bdf8; color: #000; border-color: #bae6fd; }
.u { top: 0; left: 41px; width: 42px; height: 44px; border-radius: 8px 8px 0 0; border-bottom: none; }
.dwn { bottom: 0; left: 41px; width: 42px; height: 44px; border-radius: 0 0 8px 8px; border-top: none; }
.l { left: 0; top: 41px; width: 44px; height: 42px; border-radius: 8px 0 0 8px; border-right: none; }
.r { right: 0; top: 41px; width: 44px; height: 42px; border-radius: 0 8px 8px 0; border-left: none; }
</style>
