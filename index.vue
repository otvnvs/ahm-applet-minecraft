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
        <button @click="mSel=mSel==='0'?'1':'0'">MODE: {{ mSel==='0'?'MINE':'BUILD' }}</button>
        <button @click="mv(0.4)">▲ FORWARD</button>
        <button @click="mv(-0.4)">▼ BACKWARD</button>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const g = ref(null), mSel = ref('1')
const mClrs = { '1': '#8b5a2b', '2': '#708090', '3': '#a0522d', '4': '#228b22' }
const mNames = { '0': 'Air (Mine)', '1': 'Dirt', '2': 'Stone', '3': 'Wood', '4': 'Leaves' }

let ctx, px = 4.5, py = 4.5, pa = 0, sx = 0, map = []
const mapW = 16, mapH = 16

const initWorld = () => {
  // Generate a classic 16x16 maze matrix grid
  for (let i = 0; i < mapW * mapH; i++) {
    let x = i % mapW, y = Math.floor(i / mapW)
    // Create outer boundary blocks and a few scattering inner trees/pillars
    if (x === 0 || x === mapW - 1 || y === 0 || y === mapH - 1) map[i] = '2'
    else if ((x === 5 && y === 5) || (x === 10 && y === 4)) map[i] = '3'
    else if ((x === 6 && y === 5) || (x === 10 && y === 10)) map[i] = '4'
    else map[i] = '0'
  }
}

const mv = (spd) => {
  let nx = px + Math.cos(pa) * spd
  let ny = py + Math.sin(pa) * spd
  // Basic bounding circle collision tracking loops
  if (map[Math.floor(ny) * mapW + Math.floor(nx)] === '0') { px = nx; py = ny }
  dr()
}

// Intersect targeting calculations for block mining and placement mechanics
const cc = () => {
  // Cast a forward interaction lookup ray segment vector
  let tx = Math.floor(px + Math.cos(pa) * 1.5)
  let ty = Math.floor(py + Math.sin(pa) * 1.5)
  if (tx > 0 && tx < mapW - 1 && ty > 0 && ty < mapH - 1) {
    map[ty * mapW + tx] = mSel.value
    dr()
  }
}

const ts = (e) => { if (e.touches.length) sx = e.touches.clientX }
const tm = (e) => {
  if (!e.touches.length) return
  let dx = e.touches.clientX - sx
  pa += dx * 0.012 // Horizontal swipe updates viewpoint angle layout vectors
  sx = e.touches.clientX
  dr()
}
const te = () => { sx = 0 }

// Mathematical 3D Raycasting Slice Render Projection Loop
const dr = () => {
  // Draw Sky & Ground backdrop half slices splits
  ctx.fillStyle = '#bae6fd'; ctx.fillRect(0, 0, 320, 120)
  ctx.fillStyle = '#222'; ctx.fillRect(0, 120, 320, 120)

  // Cast individual camera column rays (320 independent loops)
  for (let x = 0; x < 320; x++) {
    let rAngle = (pa - 0.5) + (x / 320)
    let rx = px, ry = py
    let sin = Math.sin(rAngle), cos = Math.cos(rAngle)
    let d = 0, hit = '0', side = 0

    // Advance ray segment coordinates across the map data array limits
    while (d < 16) {
      rx += cos * 0.05; ry += sin * 0.05; d += 0.05
      let cell = map[Math.floor(ry) * mapW + Math.floor(rx)]
      if (cell && cell !== '0') { hit = cell; break }
    }

    if (hit !== '0') {
      // Calculate fish-eye corrected geometric vertical slice line coordinates
      let wallH = Math.min(240, Math.floor(180 / (d * Math.cos(rAngle - pa))))
      let y0 = Math.max(0, 120 - wallH / 2)
      
      // Apply clean monochromatic shading falloffs matching distance values
      ctx.fillStyle = mClrs[hit] || '#708090'
      ctx.globalAlpha = Math.max(0.2, 1 - (d / 10))
      ctx.fillRect(x, y0, 1, wallH)
      ctx.globalAlpha = 1.0
    }
  }
}

onMounted(() => {
  ctx = g.value.getContext('2d')
  initWorld(); dr()
  window.addEventListener('keydown', e => {
    if (e.key === 'w' || e.key === 'ArrowUp') mv(0.35)
    if (e.key === 's' || e.key === 'ArrowDown') mv(-0.35)
    if (e.key === 'a' || e.key === 'ArrowLeft') { pa -= 0.15; dr() }
    if (e.key === 'd' || e.key === 'ArrowRight') { pa += 0.15; dr() }
  })
})
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 16px; box-sizing: border-box; font-family: sans-serif; user-select: none; overflow: hidden; }
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 18px; text-transform: uppercase; letter-spacing: 1.5px; color: #e2e8f0; } p { margin: 4px 0 0 0; color: #71717a; font-size: 12px; } b { color: #fff; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #1c1c1e; background: #222; overflow: hidden; }
canvas { display: block; width: 100%; height: 100%; image-rendering: pixelated; }
.cross { position: absolute; top: 50%; left: 50%; width: 6px; height: 6px; background: #fff; transform: translate(-50%, -50%); pointer-events: none; opacity: 0.6; mix-blend-mode: difference; }
.f { width: 100%; max-width: 320px; display: flex; flex-direction: column; gap: 8px; padding-top: 10px; }
.inv { display: flex; justify-content: center; gap: 6px; }
.inv button { width: 26px; height: 26px; border: 1px solid #2c2c2e; cursor: pointer; border-radius: 0; }
.inv button.e { border: 2px solid #fff; transform: scale(1.15); }
.d { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; width: 100%; }
button { background: #1c1c1e; border: 1px solid #2c2c2e; color: #a1a1aa; padding: 12px 2px; font-weight: 700; cursor: pointer; border-radius: 0; font-size: 10px; text-transform: uppercase; letter-spacing: 0.5px; }
button:active { background: #2c2c2e; color: #fff; }
</style>
