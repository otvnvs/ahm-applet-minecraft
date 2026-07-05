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
        <!-- Mobile Gaming Layout: D-Pad with Turning on Horizontal and Translation on Vertical -->
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

const g = ref(null), mSel = ref('1'), dpBox = ref(null)
const mClrs = { '1': '#8b5a2b', '2': '#708090', '3': '#a0522d', '4': '#228b22' }
const mNames = { '0': 'Air (Mine)', '1': 'Dirt', '2': 'Stone', '3': 'Wood', '4': 'Leaves' }

let ctx, tLoop, px = 4.5, py = 4.5, pa = 0, sx = 0, sy = 0, map = []
let dyVector = 0, dxVector = 0
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

// Fixed D-Pad Trigger Vector Configuration Map
const vPad = (dir, val) => {
  if (dir === 'u') dyVector = val * 0.065   // Move Forward
  if (dir === 'd') dyVector = val * -0.065  // Move Backward
  if (dir === 'l') dxVector = val * -0.038  // Look/Turn Left
  if (dir === 'r') dxVector = val * 0.038   // Look/Turn Right
}

// 3D Scene Drag Matrix: Left/Right drags turn angle, Up/Down drags move forward/backward
const ts = (e) => { 
  if (!e.touches.length) return
  const t = e.touches[0]
  if (!dpBox.value || t.clientX < dpBox.value.getBoundingClientRect().left - 20) {
    sx = t.clientX; sy = t.clientY 
  }
}
const tm = (e) => {
  if ((!sx && !sy) || !e.touches.length) return
  const t = e.touches[0]
  
  let dx = t.clientX - sx
  let dy = t.clientY - sy
  
  // Horizontal drag on view pane changes turning look direction angle
  pa += dx * 0.009 
  
  // Vertical drag on view pane triggers forward/backward motion translation
  let dragSpeed = dy * -0.018 
  let nx = px + Math.cos(pa) * dragSpeed
  let ny = py + Math.sin(pa) * dragSpeed
  if (map[Math.floor(ny) * mapW + Math.floor(nx)] === '0') { px = nx; py = ny }

  sx = t.clientX; sy = t.clientY
}
const te = () => { sx = 0; sy = 0 }

const cc = () => {
  let tx = Math.floor(px + Math.cos(pa) * 1.5), ty = Math.floor(py + Math.sin(pa) * 1.5)
  if (tx > 0 && tx < mapW - 1 && ty > 0 && ty < mapH - 1) map[ty * mapW + tx] = mSel.value
}

const tick = () => {
  if (dxVector !== 0) pa += dxVector // Apply continuous turning rotation

  if (dyVector !== 0) {
    let nx = px + Math.cos(pa) * dyVector
    let ny = py + Math.sin(pa) * dyVector
    if (map[Math.floor(ny) * mapW + Math.floor(nx)] === '0') { px = nx; py = ny }
  }

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
    if (e.key === 'w' || e.key === 'ArrowUp') dyVector = 0.065
    if (e.key === 's' || e.key === 'ArrowDown') dyVector = -0.065
    if (e.key === 'a' || e.key === 'ArrowLeft') dxVector = -0.038
    if (e.key === 'd' || e.key === 'ArrowRight') dxVector = 0.038
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
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 18px; text-transform: uppercase; color: #e2e8f0; } p { margin: 2px 0 0 0; color: #71717a; font-size: 11px; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #1c1c1e; background: #222; overflow: hidden; }
canvas { display: block; width: 100%; height: 100%; image-rendering: pixelated; }
.cross { position: absolute; top: 50%; left: 50%; width: 6px; height: 6px; background: #fff; transform: translate(-50%, -50%); pointer-events: none; opacity: 0.5; mix-blend-mode: difference; }

.f { width: 100%; max-width: 320px; display: flex; flex-direction: column; gap: 8px; padding-top: 4px; }
.inv { display: flex; justify-content: center; gap: 6px; }
.inv button { width: 28px; height: 28px; border: 1px solid #2c2c2e; cursor: pointer; }
.inv button.e { border: 2px solid #fff; transform: scale(1.1); }

.d { display: flex; align-items: center; width: 100%; height: 130px; }
.mode-btn { background: #1c1c1e; border: 1px solid #2c2c2e; color: #a1a1aa; padding: 14px 12px; font-weight: 700; font-size: 10px; text-transform: uppercase; cursor: pointer; }
.pad-spacer { flex: 1; }

.dpad { position: relative; width: 124px; height: 124px; display: flex; align-items: center; justify-content: center; background: rgba(24, 24, 27, 0.3); border-radius: 50%; }
.center-cap { width: 42px; height: 42px; background: #18181b; border: 1px solid #27272a; z-index: 2; pointer-events: none; grid-area: center; box-shadow: inset 0 0 8px #000; }
.p-btn { position: absolute; background: #27272a; border: 2px solid #3f3f46; color: #f4f4f5; display: flex; align-items: center; justify-content: center; font-size: 16px; font-weight: 900; cursor: pointer; touch-action: none; z-index: 3; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.5); }
.p-btn:active { background: #ef4444; color: #fff; border-color: #f43f5e; }
.u { top: 0; left: 41px; width: 42px; height: 44px; border-radius: 8px 8px 0 0; border-bottom: none; }
.dwn { bottom: 0; left: 41px; width: 42px; height: 44px; border-radius: 0 0 8px 8px; border-top: none; }
.l { left: 0; top: 41px; width: 44px; height: 42px; border-radius: 8px 0 0 8px; border-right: none; }
.r { right: 0; top: 41px; width: 44px; height: 42px; border-radius: 0 8px 8px 0; border-left: none; }
</style>
