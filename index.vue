<template>
  <div class="a" @touchstart="ts" @touchmove="tm" @touchend="te">
    <header class="b">
      <h1>RayCraft 3D</h1>
      <p>Tool: <b :style="`color:${mClrs[mSel]}`">{{ mNames[mSel] }}</b></p>
    </header>

    <main class="c">
      <canvas ref="g" width="320" height="240" @click="cc"></canvas>
      <div class="cross"></div>
    </main>

    <footer class="f">
      <!-- Material Selection Inventory Bar -->
      <div class="inv">
        <button v-for="(col, idx) in mClrs" :key="idx" @click="mSel=idx" :class="{e:mSel==idx}" :style="`background:${col}`"></button>
      </div>
      <!-- Bounded Control Layout Sheets -->
      <div class="d">
        <button @click="mSel=mSel==='0'?'1':'0'" class="mode-btn">MODE: {{ mSel==='0'?'MINE':'BUILD' }}</button>
        <div class="pad-spacer"></div>
        <!-- Upgraded RHS Dual-Axis Touch Joystick Ring Container -->
        <div class="ctrl-pad" ref="rPad" @touchstart.prevent="tpadStart" @touchmove.prevent="tpadMove" @touchend.prevent="tpadEnd">
          <div class="btn-inner">MOVE</div>
        </div>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const g = ref(null), mSel = ref('1'), rPad = ref(null)
const mClrs = { '1': '#8b5a2b', '2': '#708090', '3': '#a0522d', '4': '#228b22' }
const mNames = { '0': 'Air (Mine)', '1': 'Dirt', '2': 'Stone', '3': 'Wood', '4': 'Leaves' }

let ctx, tLoop, px = 4.5, py = 4.5, pa = 0, sx = 0, map = []
let dyVector = 0, dxVector = 0 // Dual-axis velocity engines mapped from the multi-directional stick
const mapW = 16, mapH = 16

const initWorld = () => {
  for (let i = 0; i < mapW * mapH; i++) {
    let x = i % mapW, y = Math.floor(i / mapW)
    if (x === 0 || x === mapW - 1 || y === 0 || y === mapH - 1) map[i] = '2'
    else if ((x === 5 && y === 5) || (x === 10 && y === 4)) map[i] = '3'
    else if ((x === 6 && y === 5) || (x === 10 && y === 10)) map[i] = '4'
    else map[i] = '0'
  }
}

// Bounded Dual-Axis Joystick Calculus Engine
const tpadStart = (evt) => { tpadMove(evt) }

const tpadMove = (evt) => {
  if (!evt.touches.length || !rPad.value) return
  const rect = rPad.value.getBoundingClientRect()
  const touch = evt.touches[0]
  
  // Calculate horizontal and vertical offsets relative to the pad's geometric center point
  const cx = rect.left + rect.width / 2
  const cy = rect.top + rect.height / 2
  
  const normX = (touch.clientX - cx) / (rect.width / 2)
  const normY = (touch.clientY - cy) / (rect.height / 2)
  
  // Track continuous multi-directional vectors cleanly inside baseline limits
  dxVector = Math.max(-1, Math.min(1, normX)) * 0.04  // Rotation velocity factor
  dyVector = Math.max(-1, Math.min(1, normY)) * -0.08 // Forward/Backward velocity factor
}

const tpadEnd = () => { dxVector = 0; dyVector = 0 }

// Manual screen swipe lookup backup triggers (Fires only outside joystick boundary box handles)
const ts = (e) => { 
  if (e.touches.length && (!rPad.value || e.touches[0].clientX < rPad.value.getBoundingClientRect().left)) {
    sx = e.touches[0].clientX 
  }
}
const tm = (e) => {
  if (!sx || !e.touches.length) return
  let dx = e.touches[0].clientX - sx
  pa += dx * 0.012
  sx = e.touches[0].clientX
}
const te = () => { sx = 0 }

// Direct Screen Center Voxel Mining Interaction Pipeline
const cc = () => {
  let tx = Math.floor(px + Math.cos(pa) * 1.5)
  let ty = Math.floor(py + Math.sin(pa) * 1.5)
  if (tx > 0 && tx < mapW - 1 && ty > 0 && ty < mapH - 1) {
    map[ty * mapW + tx] = mSel.value
  }
}

// Global Core Engine Tick Update Frame loop
const tick = () => {
  // Apply real-time angular updates from horizontal joystick deflection
  if (dxVector !== 0) {
    pa += dxVector
  }

  // Apply real-time positional updates from vertical joystick deflection
  if (dyVector !== 0) {
    let nx = px + Math.cos(pa) * dyVector
    let ny = py + Math.sin(pa) * dyVector
    if (map[Math.floor(ny) * mapW + Math.floor(nx)] === '0') { px = nx; py = ny }
  }

  // Clear canvas view pane and process pseudo-3D raycast slice draws
  ctx.fillStyle = '#bae6fd'; ctx.fillRect(0, 0, 320, 120)
  ctx.fillStyle = '#222'; ctx.fillRect(0, 120, 320, 120)

  for (let x = 0; x < 320; x++) {
    let rAngle = (pa - 0.5) + (x / 320)
    let rx = px, ry = py
    let sin = Math.sin(rAngle), cos = Math.cos(rAngle)
    let d = 0, hit = '0'

    while (d < 16) {
      rx += cos * 0.05; ry += sin * 0.05; d += 0.05
      let cell = map[Math.floor(ry) * mapW + Math.floor(rx)]
      if (cell && cell !== '0') { hit = cell; break }
    }

    if (hit !== '0') {
      let wallH = Math.min(240, Math.floor(180 / (d * Math.cos(rAngle - pa))))
      let y0 = Math.max(0, 120 - wallH / 2)
      
      ctx.fillStyle = mClrs[hit] || '#708090'
      ctx.globalAlpha = Math.max(0.2, 1 - (d / 10))
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
    if (e.key === 'w' || e.key === 'ArrowUp') dyVector = 0.08
    if (e.key === 's' || e.key === 'ArrowDown') dyVector = -0.08
    if (e.key === 'a' || e.key === 'ArrowLeft') dxVector = -0.04
    if (e.key === 'd' || e.key === 'ArrowRight') dxVector = 0.04
  })
  window.addEventListener('keyup', e => {
    if (['w', 's', 'ArrowUp', 'ArrowDown'].includes(e.key)) dyVector = 0
    if (['a', 'd', 'ArrowLeft', 'ArrowRight'].includes(e.key)) dxVector = 0
  })
})

onUnmounted(() => clearInterval(tLoop))
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 16px; box-sizing: border-box; font-family: sans-serif; user-select: none; overflow: hidden; }
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 20px; text-transform: uppercase; letter-spacing: 1.5px; color: #e2e8f0; } p { margin: 4px 0 0 0; color: #71717a; font-size: 12px; } b { color: #fff; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #1c1c1e; background: #222; overflow: hidden; }
canvas { display: block; width: 100%; height: 100%; image-rendering: pixelated; }
.cross { position: absolute; top: 50%; left: 50%; width: 6px; height: 6px; background: #fff; transform: translate(-50%, -50%); pointer-events: none; opacity: 0.6; mix-blend-mode: difference; }

.f { width: 100%; max-width: 320px; display: flex; flex-direction: column; gap: 8px; padding-top: 10px; }
.inv { display: flex; justify-content: center; gap: 6px; }
.inv button { width: 26px; height: 26px; border: 1px solid #2c2c2e; cursor: pointer; border-radius: 0; }
.inv button.e { border: 2px solid #fff; transform: scale(1.15); }

/* Bottom Toolbar Overlay Layouts featuring the fully functional 2D joystick module */
.d { display: flex; align-items: center; width: 100%; height: 76px; }
.mode-btn { background: #1c1c1e; border: 1px solid #2c2c2e; color: #a1a1aa; padding: 12px 14px; font-weight: 700; cursor: pointer; border-radius: 0; font-size: 11px; text-transform: uppercase; letter-spacing: 0.5px; }
.mode-btn:active { background: #2c2c2e; color: #fff; }
.pad-spacer { flex: 1; }

.ctrl-pad { width: 72px; height: 72px; background: #111112; border: 2px solid #2c2c2e; border-radius: 50%; display: flex; align-items: center; justify-content: center; touch-action: none; cursor: pointer; }
.ctrl-pad:active { background: #1c1c1e; border-color: #ef4444; }
.btn-inner { font-size: 9px; font-weight: 800; color: #71717a; letter-spacing: 0.5px; text-transform: uppercase; pointer-events: none; }
.ctrl-pad:active .btn-inner { color: #fff; }
</style>
