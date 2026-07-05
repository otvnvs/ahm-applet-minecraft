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
      <div class="inv">
        <button v-for="(col, idx) in mClrs" :key="idx" @click="mSel=idx" :class="{e:mSel==idx}" :style="`background:${col}`"></button>
      </div>
      <div class="d">
        <button @click="mSel=mSel==='0'?'1':'0'" class="mode-btn">MODE: {{ mSel==='0'?'MINE':'BUILD' }}</button>
        <div class="pad-spacer"></div>
        <!-- Compact D-Pad / Joystick Mesh Cluster Layout -->
        <div class="pad-cluster">
          <button class="p-btn u" @touchstart.prevent="vPad('u',1)" @touchend.prevent="vPad('u',0)">▲</button>
          <button class="p-btn l" @touchstart.prevent="vPad('l',1)" @touchend.prevent="vPad('l',0)">◀</button>
          <div class="ctrl-pad" ref="rPad" @touchstart.prevent="tpm" @touchmove.prevent="tpm" @touchend.prevent="tpe">
            <div class="btn-inner">MOVE</div>
          </div>
          <button class="p-btn r" @touchstart.prevent="vPad('r',1)" @touchend.prevent="vPad('r',0)">▶</button>
          <button class="p-btn dwn" @touchstart.prevent="vPad('d',1)" @touchend.prevent="vPad('d',0)">▼</button>
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

let ctx, tLoop, px = 4.5, py = 4.5, pa = 0, sx = 0, sy = 0, map = []
let dyVector = 0, dxVector = 0, strafeV = 0 // Dual-axis velocity engines
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

// Bounded Dual-Axis Joystick Engine: Continuous Mapping to 4-Direction Axis Translation Vector
const tpm = (e) => {
  if (!e.touches.length || !rPad.value) return
  const rect = rPad.value.getBoundingClientRect()
  const touch = e.touches[0]
  const normX = (touch.clientX - (rect.left + rect.width / 2)) / (rect.width / 2)
  const normY = (touch.clientY - (rect.top + rect.height / 2)) / (rect.height / 2)
  strafeV = Math.max(-1, Math.min(1, normX)) * 0.06 // Horizontal maps to Strafe Left/Right Vector
  dyVector = Math.max(-1, Math.min(1, normY)) * -0.06 // Vertical maps to Forward/Backward Vector
}
const tpe = () => { strafeV = 0; dyVector = 0 }

// Surrounding D-Pad Fixed-Trigger Engine for Digital Turning and Translation Keys
const vPad = (dir, val) => {
  if (dir === 'u') dyVector = val * 0.06
  if (dir === 'd') dyVector = val * -0.06
  if (dir === 'l') dxVector = val * -0.035 // Dedicated Turning Left Engine
  if (dir === 'r') dxVector = val * 0.035  // Dedicated Turning Right Engine
}

// Global Touch drag configuration framework (Drag outside joystick maps to rotation / standard movement translation seamlessly)
const ts = (e) => { 
  if (!e.touches.length) return
  const touch = e.touches[0]
  if (!rPad.value || touch.clientX < rPad.value.getBoundingClientRect().left - 40) {
    sx = touch.clientX; sy = touch.clientY 
  }
}
const tm = (e) => {
  if ((!sx && !sy) || !e.touches.length) return
  const touch = e.touches[0]
  let dx = touch.clientX - sx
  let dy = touch.clientY - sy
  
  pa += dx * 0.008 // Horizontal touch drag updates camera gaze direction angle
  let dragSpeed = dy * -0.0015
  let nx = px + Math.cos(pa) * dragSpeed
  let ny = py + Math.sin(pa) * dragSpeed
  if (map[Math.floor(ny) * mapW + Math.floor(nx)] === '0') { px = nx; py = ny }

  sx = touch.clientX; sy = touch.clientY
}
const te = () => { sx = 0; sy = 0 }

// Voxel mining point execution
const cc = () => {
  let tx = Math.floor(px + Math.cos(pa) * 1.5), ty = Math.floor(py + Math.sin(pa) * 1.5)
  if (tx > 0 && tx < mapW - 1 && ty > 0 && ty < mapH - 1) map[ty * mapW + tx] = mSel.value
}

const tick = () => {
  if (dxVector !== 0) pa += dxVector // Rotational Delta

  // 4-Direction Cartesian Physics Vector Step Controller (Combines Front/Back and Strafe Translation)
  let mx = (dyVector !== 0 ? Math.cos(pa) * dyVector : 0) + (strafeV !== 0 ? Math.cos(pa + Math.PI / 2) * strafeV : 0)
  let my = (dyVector !== 0 ? Math.sin(pa) * dyVector : 0) + (strafeV !== 0 ? Math.sin(pa + Math.PI / 2) * strafeV : 0)
  
  let nx = px + mx, ny = py + my
  if (map[Math.floor(ny) * mapW + Math.floor(nx)] === '0') { px = nx; py = ny }

  // Clear render engine setup buffer targets
  ctx.fillStyle = '#bae6fd'; ctx.fillRect(0, 0, 320, 120)
  ctx.fillStyle = '#222'; ctx.fillRect(0, 120, 320, 120)

  for (let x = 0; x < 320; x++) {
    let rAngle = (pa - 0.5) + (x / 320), rx = px, ry = py
    let sin = Math.sin(rAngle), cos = Math.cos(rAngle), d = 0, hit = '0'

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
    if (e.key === 'w' || e.key === 'ArrowUp') dyVector = 0.06
    if (e.key === 's' || e.key === 'ArrowDown') dyVector = -0.06
    if (e.key === 'a') strafeV = -0.06
    if (e.key === 'd') strafeV = 0.06
    if (e.key === 'ArrowLeft') dxVector = -0.035
    if (e.key === 'ArrowRight') dxVector = 0.035
  })
  window.addEventListener('keyup', e => {
    if (['w', 's', 'ArrowUp', 'ArrowDown'].includes(e.key)) dyVector = 0
    if (['a', 'd'].includes(e.key)) strafeV = 0
    if (['ArrowLeft', 'ArrowRight'].includes(e.key)) dxVector = 0
  })
})
onUnmounted(() => clearInterval(tLoop))
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 16px; box-sizing: border-box; font-family: sans-serif; user-select: none; overflow: hidden; }
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 20px; text-transform: uppercase; letter-spacing: 1.5px; color: #e2e8f0; } p { margin: 4px 0 0 0; color: #71717a; font-size: 12px; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #1c1c1e; background: #222; overflow: hidden; }
canvas { display: block; width: 100%; height: 100%; image-rendering: pixelated; }
.cross { position: absolute; top: 50%; left: 50%; width: 6px; height: 6px; background: #fff; transform: translate(-50%, -50%); pointer-events: none; opacity: 0.6; mix-blend-mode: difference; }

.f { width: 100%; max-width: 320px; display: flex; flex-direction: column; gap: 8px; padding-top: 10px; }
.inv { display: flex; justify-content: center; gap: 6px; }
.inv button { width: 26px; height: 26px; border: 1px solid #2c2c2e; cursor: pointer; }
.inv button.e { border: 2px solid #fff; transform: scale(1.15); }

.d { display: flex; align-items: center; width: 100%; height: 110px; }
.mode-btn { background: #1c1c1e; border: 1px solid #2c2c2e; color: #a1a1aa; padding: 10px 12px; font-weight: 700; cursor: pointer; font-size: 10px; text-transform: uppercase; }
.pad-spacer { flex: 1; }

/* D-Pad Composite Cluster Shell Layout */
.pad-cluster { position: relative; width: 110px; height: 110px; display: flex; align-items: center; justify-content: center; }
.ctrl-pad { z-index: 2; width: 44px; height: 44px; background: #111112; border: 2px solid #3f3f46; border-radius: 50%; display: flex; align-items: center; justify-content: center; touch-action: none; }
.ctrl-pad:active { border-color: #ef4444; background: #1c1c1e; }
.btn-inner { font-size: 8px; font-weight: 800; color: #71717a; pointer-events: none; }
.p-btn { position: absolute; width: 30px; height: 30px; background: #18181b; border: 1px solid #3f3f46; color: #a1a1aa; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: bold; cursor: pointer; touch-action: none; z-index: 1; }
.p-btn:active { background: #ef4444; color: #fff; border-color: #f43f5e; }
.u { top: 0; left: 40px; }
.dwn { bottom: 0; left: 40px; }
.l { left: 0; top: 40px; }
.r { right: 0; top: 40px; }
</style>
