<template>
  <div class="a" @touchstart="ts" @touchmove="tm" @touchend="te">
    <header class="b">
      <h1>Vox3D</h1>
      <p>Mat: <b :style="`color:${mClrs[mSel]}`">{{ mNames[mSel] }}</b></p>
    </header>

    <main class="c">
      <canvas ref="g" width="320" height="240" @click="cc"></canvas>
      <div class="ret"></div>
    </main>

    <footer class="f">
      <div class="inv">
        <button v-for="(c, i) in mClrs" :key="i" @click="mSel=i" :class="{e:mSel==i}" :style="`background:${c}`"></button>
      </div>
      <div class="d">
        <button @click="mv(0, -0.4)">▲ MOVE</button>
        <button @click="mSel=mSel==='1'?'0':'1'">MODE: {{ mSel==='0'?'MINE':'BUILD' }}</button>
        <button @click="mv(0, 0.4)">▼ BACK</button>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const g = ref(null), mSel = ref('1')
const mClrs = { '1': '#8b5a2b', '2': '#708090', '3': '#a0522d', '4': '#228b22' }
const mNames = { '0': 'Air (Mine)', '1': 'Dirt', '2': 'Stone', '3': 'Wood', '4': 'Leaves' }

let ctx, t, voxels = {}, cx = 0, cy = 2, cz = 5, ry = 0, rx = 0, sx = 0, sy = 0

// Native 3D Rotation and Perspective Projection Pipeline Engine
const project = (x, y, z) => {
  // Translate relative to viewport camera coordinates positions
  let dx = x - cx, dy = y - cy, dz = z - cz

  // Apply horizontal Camera Rotation Matrix (Y-Axis)
  let cosY = Math.cos(ry), sinY = Math.sin(ry)
  let x1 = dx * cosY - dz * sinY
  let z1 = dx * sinY + dz * cosY

  // Apply vertical Camera Rotation Matrix (X-Axis)
  let cosX = Math.cos(rx), sinX = Math.sin(rx)
  let y2 = dy * cosX - z1 * sinX
  let z2 = dy * sinX + z1 * cosX

  if (z2 <= 0.1) return null // Clip entities behind view pane camera array

  // Core geometric division calculations projecting vectors onto 2D viewport bounds
  let fov = 180
  return {
    x: 160 + (x1 * fov) / z2,
    y: 120 + (y2 * fov) / z2,
    z: z2
  }
}

// Generates procedural 3D wireframe sandbox voxels map layouts
const initWorld = () => {
  for (let x = -4; x <= 4; x++) {
    for (let z = -4; z <= 4; z++) {
      let h = Math.floor(Math.sin(x * 0.6) * 1.2)
      voxels[`${x},${h},${z}`] = '1'
      voxels[`${x},${h-1},${z}`] = '2'
    }
  }
}

const mv = (f, s) => {
  cx += Math.sin(ry) * s; cz += Math.cos(ry) * s
  cy += Math.sin(rx) * f; cz += Math.cos(rx) * f
  dr()
}

// Screen Taps Mining / Constructive Node Intersect calculation arrays
const cc = () => {
  // Cast simple forward ray tracing indices vectors directly matching center screen crosshair
  let vx = Math.round(cx - Math.sin(ry) * 2.5)
  let vy = Math.round(cy - Math.sin(rx) * 2.5)
  let vz = Math.round(cz - Math.cos(ry) * 2.5)
  const k = `${vx},${vy},${vz}`
  
  if (mSel.value === '0') delete voxels[k]
  else voxels[k] = mSel.value
  dr()
}

const ts = (e) => { if (e.touches.length) { sx = e.touches[0].clientX; sy = e.touches[0].clientY } }
const tm = (e) => {
  if (!e.touches.length) return
  let dx = e.touches[0].clientX - sx, dy = e.touches[0].clientY - sy
  ry -= dx * 0.008; rx = Math.max(-0.8, Math.min(0.8, rx - dy * 0.008))
  sx = e.touches[0].clientX; sy = e.touches[0].clientY
  dr()
}
const te = () => { sx = 0; sy = 0 }

// Canvas Polyfill Rendering Engine Loop
const dr = () => {
  ctx.fillStyle = '#bae6fd'; ctx.fillRect(0, 0, 320, 240)

  // Sort spatial indexes to avoid graphic overlay breaks
  let list = []
  Object.keys(voxels).forEach(k => {
    let [x, y, z] = k.split(',').map(Number)
    let p = project(x, y, z)
    if (p) list.push({ x, y, z, id: voxels[k], depth: p.z })
  })
  list.sort((a, b) => b.depth - a.depth)

  // Render individual wireframe 3D block isometric hulls
  list.forEach(b => {
    let p1 = project(b.x - 0.4, b.y - 0.4, b.z - 0.4)
    let p2 = project(b.x + 0.4, b.y - 0.4, b.z - 0.4)
    let p3 = project(b.x + 0.4, b.y + 0.4, b.z - 0.4)
    let p4 = project(b.x - 0.4, b.y + 0.4, b.z - 0.4)
    let p5 = project(b.x - 0.4, b.y - 0.4, b.z + 0.4)
    let p6 = project(b.x + 0.4, b.y - 0.4, b.z + 0.4)
    let p7 = project(b.x + 0.4, b.y + 0.4, b.z + 0.4)
    let p8 = project(b.x - 0.4, b.y + 0.4, b.z + 0.4)

    if (!p1 || !p2 || !p3 || !p4 || !p5 || !p6 || !p7 || !p8) return

    // Draw primary shaded solid visual faces
    ctx.fillStyle = mClrs[b.id] || '#8b5a2b'
    ctx.beginPath()
    ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y); ctx.lineTo(p3.x, p3.y); ctx.lineTo(p4.x, p4.y)
    ctx.fill()

    ctx.beginPath()
    ctx.moveTo(p5.x, p5.y); ctx.lineTo(p6.x, p6.y); ctx.lineTo(p7.x, p7.y); ctx.lineTo(p8.x, p8.y)
    ctx.fill()

    // Draw structural clean grid line paths borders overrides
    ctx.strokeStyle = 'rgba(0,0,0,0.25)'
    ctx.lineWidth = 1
    ctx.beginPath()
    ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y); ctx.lineTo(p3.x, p3.y); ctx.lineTo(p4.x, p4.y); ctx.closePath()
    ctx.moveTo(p5.x, p5.y); ctx.lineTo(p6.x, p6.y); ctx.lineTo(p7.x, p7.y); ctx.lineTo(p8.x, p8.y); ctx.closePath()
    ctx.lineTo(p4.x, p4.y); ctx.moveTo(p6.x, p6.y); ctx.lineTo(p2.x, p2.y); ctx.moveTo(p7.x, p7.y); ctx.lineTo(p3.x, p3.y)
    ctx.stroke()
  })
}

onMounted(() => {
  ctx = g.value.getContext('2d')
  initWorld(); dr()
  window.addEventListener('keydown', e => {
    if (e.key === 'w') mv(0, -0.3); if (e.key === 's') mv(0, 0.3)
    if (e.key === 'a') { ry -= 0.1; dr() }; if (e.key === 'd') { ry += 0.1; dr() }
  })
})
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 12px; box-sizing: border-box; font-family: sans-serif; user-select: none; overflow: hidden; }
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 18px; text-transform: uppercase; letter-spacing: 1px; } p { margin: 2px; color: #71717a; font-size: 11px; } b { color: #fff; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #1c1c1e; background: #bae6fd; }
canvas { display: block; width: 100%; height: 100%; }
.ret { position: absolute; top: 50%; left: 50%; width: 4px; height: 4px; background: #ff0000; transform: translate(-50%, -50%); pointer-events: none; opacity: 0.8; }
.f { width: 100%; max-width: 320px; display: flex; flex-direction: column; gap: 8px; padding: 4px 0; }
.inv { display: flex; justify-content: center; gap: 4px; }
.inv button { width: 24px; height: 24px; border: 1px solid #2c2c2e; cursor: pointer; border-radius: 0; }
.inv button.e { border: 2px solid #fff; transform: scale(1.1); }
.d { display: grid; grid-template-columns: repeat(3, 1fr); gap: 4px; width: 100%; }
button { background: #111; border: 1px solid #2c2c2e; color: #a1a1aa; padding: 10px 4px; font-weight: 700; cursor: pointer; border-radius: 0; font-size: 10px; text-transform: uppercase; }
button:active { background: #1c1c1e; color: #fff; }
</style>
