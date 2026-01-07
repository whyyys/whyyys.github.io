<template>
  <canvas ref="canvas" class="grid-canvas"></canvas>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const canvas = ref<HTMLCanvasElement>()
let ctx: CanvasRenderingContext2D
let raf = 0

/* ---------- 网格参数 ---------- */
const edge = 26            // 三角形边长（像素）
const hEdge = edge * 0.866 // 等高三角形高度
const grid: Triangle[] = []
const dots: Dot[] = []

class Dot {
  x: number; y: number
  glow = 0                // 0-1 发光强度
  constructor (x: number, y: number) { this.x = x; this.y = y }
}

class Triangle {
  a: Dot; b: Dot; c: Dot
  constructor (a: Dot, b: Dot, c: Dot) { this.a = a; this.b = b; this.c = c }
  /* 计算重心到鼠标距离 */
  dist (mx: number, my: number) {
    const cx = (this.a.x + this.b.x + this.c.x) / 3
    const cy = (this.a.y + this.b.y + this.c.y) / 3
    return Math.hypot(cx - mx, cy - my)
  }
}

/* ---------- 生成整个屏幕的等边三角网格 ---------- */
function buildGrid () {
  grid.length = 0
  dots.length = 0
  const cols = Math.ceil(innerWidth / edge) + 2
  const rows = Math.ceil(innerHeight / hEdge) + 2
  /* 先建点阵 */
  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const x = col * edge + (row % 2 ? edge / 2 : 0)
      const y = row * hEdge
      dots.push(new Dot(x, y))
    }
  }
  /* 再连三角形 */
  for (let row = 0; row < rows - 1; row++) {
    for (let col = 0; col < cols - 1; col++) {
      const i = row * cols + col
      const up = row % 2
      if (up) {
        grid.push(new Triangle(dots[i], dots[i + 1], dots[i + cols]))
        grid.push(new Triangle(dots[i + 1], dots[i + cols], dots[i + cols + 1]))
      } else {
        grid.push(new Triangle(dots[i], dots[i + cols], dots[i + cols + 1]))
        grid.push(new Triangle(dots[i], dots[i + 1], dots[i + cols + 1]))
      }
    }
  }
}

/* ---------- 动画循环 ---------- */
let mouse = { x: -999, y: -999 }

function loop () {
  /* 不填充背景 → 保留透明 */
  ctx.clearRect(0, 0, innerWidth, innerHeight)

  /* 1. 更新 glow */
  const maxDist = 120
  for (const t of grid) {
    const d = t.dist(mouse.x, mouse.y)
    const target = d < maxDist ? 1 - d / maxDist : 0
    // 平滑逼近
    t.a.glow += (target - t.a.glow) * 0.08
    t.b.glow += (target - t.b.glow) * 0.08
    t.c.glow += (target - t.c.glow) * 0.08
  }

  /* 2. 画连线（发光越强越亮） */
  ctx.lineWidth = 1.2
  for (const t of grid) {
    const avgGlow = (t.a.glow + t.b.glow + t.c.glow) / 3
    if (avgGlow < 0.05) continue
    const color = `rgba(65,209,255,${avgGlow * 0.9})`
    drawLine(t.a, t.b, color)
    drawLine(t.b, t.c, color)
    drawLine(t.c, t.a, color)
  }

  /* 3. 画顶点 */
  for (const d of dots) {
    if (d.glow < 0.05) continue
    ctx.beginPath()
    ctx.arc(d.x, d.y, 2 + d.glow * 1.5, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(255,255,255,${d.glow})`
    ctx.fill()
  }

  raf = requestAnimationFrame(loop)
}

function drawLine (a: Dot, b: Dot, color: string) {
  ctx.beginPath()
  ctx.moveTo(a.x, a.y)
  ctx.lineTo(b.x, b.y)
  ctx.strokeStyle = color
  ctx.stroke()
}

/* ---------- 生命周期 ---------- */
onMounted(() => {
  const el = canvas.value!
  ctx = el.getContext('2d')!
  resize()
  window.addEventListener('resize', resize)
  window.addEventListener('mousemove', (e) => {
    mouse.x = e.clientX
    mouse.y = e.clientY
  })
  loop()
})

onUnmounted(() => {
  cancelAnimationFrame(raf)
  window.removeEventListener('resize', resize)
})

function resize () {
  const el = canvas.value!
  el.width = innerWidth * devicePixelRatio
  el.height = innerHeight * devicePixelRatio
  ctx.setTransform(1, 0, 0, 1, 0, 0)
  ctx.scale(devicePixelRatio, devicePixelRatio)
  buildGrid()
}
</script>

<style scoped>
.grid-canvas {
  position: fixed;
  inset: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: -1;
}
</style>