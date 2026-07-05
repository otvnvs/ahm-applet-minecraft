<template>
  <div class="a">
    <header class="b">
      <h1>MiniCraft</h1>
      <p>Pos: <b>X:{{ Math.round(px) }} Y:{{ Math.round(py) }}</b></p>
    </header>

    <main class="c" @click="cc">
      <canvas ref="g" width="320" height="240"></canvas>
    </main>

    <!-- Lower Device Inventory Bar & Motion Controllers -->
    <footer class="f">
      <div class="inv">
        <button v-for="(c, m) in clrs" :key="m" @click="t=m" :class="{e:t==m}" :style="`background:${c}`"></button>
      </div>
      <div class="d">
        <button @click="m(-1,0)">◀</button>
        <button @click="m(0,-1)">▲</button>
        <button @click="m(0,1)">▼</button>
        <button @click="m(1,0)">▶</button>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const g = ref(null), t = ref('1'), clrs = { '0': '#000', '1': '#8b5a2b', '2': '#708090', '3': '#a0522d', '4': '#228b22' }
let ctx, cn, px = 10, py = 8, w = {}

// Generates procedural terrain block matrix columns on initialization
const initWorld = () => {
  for (let x = -50; x < 100; x++) {
    let h = 12 + Math.floor(Math.sin(x * 0.15) * 3) // Sine wave terrain height map
    for (let y = 0; y < 24; y++) {
      let b = '0'
      if (y > h) b = '2' // Stone layers
      else if (y === h) b = '1' // Topsoil Dirt
      else if (x === 15 && y === h - 1) b = '3' // Simple structural Wood tree stem
      else if (x === 15 && y === h - 2) b = '4' // Minimal Leaf block crown
      if (b !== '0') w[`${x},${y}`] = b
    }
  }
}

const m = (dx, dy) => { px += dx; py = Math.max(0, Math.min(20, py + dy)); dr() }

// Canvas Click Input Parser for Block Extraction/Placement Operations
const cc = (e) => {
  if (!cn) return
  const r = cn.getBoundingClientRect()
  const cx = Math.floor((e.clientX - r.left) / 16) + Math.floor(px) - 10
  const cy = Math.floor((e.clientY - r.top) / 16) + Math.floor(py) - 7
  const k = `${cx},${cy}`
  w[k] = w[k] ? undefined : t.value // Toggles tile existence natively
  dr()
}

// Graphic Vector Rendering Pipeline
const dr = () => {
  ctx.fillStyle = '#bae6fd'; ctx.fillRect(0, 0, 320, 240) // Sky layer background fill
  for (let x = 0; x < 20; x++) {
    for (let y = 0; y < 15; y++) {
      const bx = x + Math.floor(px) - 10, by = y + Math.floor(py) - 7
      const block = w[`${bx},${by}`]
      if (block) {
        ctx.fillStyle = clrs[block]
        ctx.fillRect(x * 16, y * 16, 16, 16)
        ctx.strokeStyle = '#00000022'; ctx.strokeRect(x * 16, y * 16, 16, 16)
      }
    }
  }
  // Draws crosshair targeting voxel box in canvas center screen frame layout
  ctx.fillStyle = '#ff0000'
  ctx.fillRect(158, 118, 4, 4)
}

onMounted(() => {
  cn = g.value; ctx = cn.getContext('2d')
  initWorld(); dr()
  window.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft') m(-1, 0); if (e.key === 'ArrowRight') m(1, 0)
    if (e.key === 'ArrowUp') m(0, -1); if (e.key === 'ArrowDown') m(0, 1)
  })
})
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 12px; box-sizing: border-box; font-family: sans-serif; user-select: none; overflow: hidden; }
.b { text-align: center; width: 100%; } h1 { margin: 0; font-size: 18px; text-transform: uppercase; letter-spacing: 1px; } p { margin: 2px; color: #71717a; font-size: 11px; } b { color: #fff; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 240; position: relative; border: 2px solid #1c1c1e; background: #bae6fd; }
canvas { display: block; width: 100%; height: 100%; }
.f { width: 100%; max-width: 320px; display: flex; flex-direction: column; gap: 8px; padding: 4px 0; }
.inv { display: flex; justify-content: center; gap: 4px; }
.inv button { width: 24px; height: 24px; border: 1px solid #2c2c2e; cursor: pointer; border-radius: 0; }
.inv button.e { border: 2px solid #fff; transform: scale(1.1); }
.d { display: grid; grid-template-columns: repeat(4, 1fr); gap: 4px; width: 180px; margin: 0 auto; }
button { background: #111; border: 1px solid #2c2c2e; color: #a1a1aa; padding: 8px; font-weight: 700; cursor: pointer; border-radius: 0; font-size: 12px; }
button:active { background: #1c1c1e; color: #fff; }
</style>
