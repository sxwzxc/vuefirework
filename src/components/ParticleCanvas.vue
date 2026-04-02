<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const canvasRef = ref<HTMLCanvasElement | null>(null)
let animationId = 0
let ctx: CanvasRenderingContext2D | null = null
let time = 0

// ========== 鼠标状态 ==========
let mouse = { x: -9999, y: -9999, isDown: false, downTime: 0 }

// ========== 配置 ==========
const CONFIG = {
  // 主粒子
  particleCount: 200,
  particleMinSize: 1,
  particleMaxSize: 3,
  particleSpeed: 0.8,
  // 背景星星
  starCount: 120,
  // 螺旋
  spiralSpeed: 0.08,
  spiralShrink: 0.988,
  spiralMinDist: 6,
  spiralInfluenceRadius: 400,
  // 连线
  connectionDistance: 130,
  connectionMaxAlpha: 0.2,
  // 鼠标悬浮吸引
  hoverRadius: 200,
  hoverForce: 0.003,
  // 拖尾
  trailLength: 6,
  trailAlphaDecay: 0.65,
}

// ========== 接口 ==========
interface Trail {
  x: number
  y: number
}

interface Particle {
  x: number
  y: number
  vx: number
  vy: number
  size: number
  baseSize: number
  color: string       // "r, g, b"
  hue: number         // HSL hue for rainbow effect
  alpha: number
  baseVx: number
  baseVy: number
  pulseOffset: number
  trail: Trail[]
  layer: number       // 0=far, 1=mid, 2=near
}

interface Star {
  x: number
  y: number
  size: number
  alpha: number
  twinkleSpeed: number
  twinkleOffset: number
}

let particles: Particle[] = []
let stars: Star[] = []

// ========== HSL 转 RGB ==========
function hslToRgb(h: number, s: number, l: number): [number, number, number] {
  s /= 100
  l /= 100
  const k = (n: number) => (n + h / 30) % 12
  const a = s * Math.min(l, 1 - l)
  const f = (n: number) => l - a * Math.max(-1, Math.min(k(n) - 3, Math.min(9 - k(n), 1)))
  return [Math.round(f(0) * 255), Math.round(f(8) * 255), Math.round(f(4) * 255)]
}

function hueToRgbStr(hue: number): string {
  const [r, g, b] = hslToRgb(hue, 85, 60)
  return `${r}, ${g}, ${b}`
}

// ========== 粒子颜色 ==========
const PALETTE_HUES = [270, 220, 190, 160, 35, 0, 330, 250]

function randomHue(): number {
  return PALETTE_HUES[Math.floor(Math.random() * PALETTE_HUES.length)] + (Math.random() - 0.5) * 20
}

// ========== 创建粒子 ==========
function createParticle(w: number, h: number, layer: number = 1): Particle {
  const speedMul = layer === 0 ? 0.3 : layer === 1 ? 0.7 : 1.0
  const sizeMul = layer === 0 ? 0.5 : layer === 1 ? 1.0 : 1.5
  const vx = (Math.random() - 0.5) * CONFIG.particleSpeed * speedMul * 2
  const vy = (Math.random() - 0.5) * CONFIG.particleSpeed * speedMul * 2
  const hue = randomHue()
  const baseSize = (Math.random() * (CONFIG.particleMaxSize - CONFIG.particleMinSize) + CONFIG.particleMinSize) * sizeMul
  return {
    x: Math.random() * w,
    y: Math.random() * h,
    vx,
    vy,
    size: baseSize,
    baseSize,
    color: hueToRgbStr(hue),
    hue,
    alpha: (Math.random() * 0.4 + 0.6) * (layer === 0 ? 0.4 : layer === 1 ? 0.7 : 1.0),
    baseVx: vx,
    baseVy: vy,
    pulseOffset: Math.random() * Math.PI * 2,
    trail: [],
    layer,
  }
}

function createStar(w: number, h: number): Star {
  return {
    x: Math.random() * w,
    y: Math.random() * h,
    size: Math.random() * 1.5 + 0.3,
    alpha: Math.random() * 0.6 + 0.2,
    twinkleSpeed: Math.random() * 0.02 + 0.005,
    twinkleOffset: Math.random() * Math.PI * 2,
  }
}

// ========== 画布初始化 ==========
function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  const dpr = window.devicePixelRatio || 1
  canvas.width = window.innerWidth * dpr
  canvas.height = window.innerHeight * dpr
  canvas.style.width = window.innerWidth + 'px'
  canvas.style.height = window.innerHeight + 'px'
  ctx = canvas.getContext('2d')
  if (ctx) ctx.scale(dpr, dpr)
}

// ========== 鼠标事件 ==========
function onMouseDown(e: MouseEvent) {
  mouse.x = e.clientX
  mouse.y = e.clientY
  mouse.isDown = true
  mouse.downTime = time
}
function onMouseUp() {
  mouse.isDown = false
}
function onMouseMove(e: MouseEvent) {
  mouse.x = e.clientX
  mouse.y = e.clientY
}
function onTouchStart(e: TouchEvent) {
  const t = e.touches[0]
  mouse.x = t.clientX
  mouse.y = t.clientY
  mouse.isDown = true
  mouse.downTime = time
}
function onTouchEnd() {
  mouse.isDown = false
}
function onTouchMove(e: TouchEvent) {
  const t = e.touches[0]
  mouse.x = t.clientX
  mouse.y = t.clientY
}

// ========== 粒子更新 ==========
function updateParticle(p: Particle) {
  const w = window.innerWidth
  const h = window.innerHeight

  // 保存拖尾
  p.trail.unshift({ x: p.x, y: p.y })
  if (p.trail.length > CONFIG.trailLength) {
    p.trail.pop()
  }

  if (mouse.isDown) {
    // === 螺旋吸引 ===
    const dx = p.x - mouse.x
    const dy = p.y - mouse.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    const angle = Math.atan2(dy, dx)

    // 螺旋强度随距离衰减
    const influence = Math.min(1, CONFIG.spiralInfluenceRadius / (dist + 1))

    // 动态旋转速度：离中心越近旋转越快
    const rotSpeed = CONFIG.spiralSpeed * (1 + 80 / (dist + 20))
    const spiralAngle = angle + rotSpeed
    const targetDist = Math.max(dist * CONFIG.spiralShrink, CONFIG.spiralMinDist)

    const targetX = mouse.x + Math.cos(spiralAngle) * targetDist
    const targetY = mouse.y + Math.sin(spiralAngle) * targetDist

    p.vx += (targetX - p.x) * 0.06 * influence
    p.vy += (targetY - p.y) * 0.06 * influence

    // 螺旋时颜色随角度和时间变化 (彩虹效果)
    p.hue = ((angle / Math.PI) * 180 + time * 2) % 360
    p.color = hueToRgbStr(p.hue)
  } else {
    // === 自由漂浮 ===
    // 缓慢恢复到基础速度
    p.vx += (p.baseVx - p.vx) * 0.015
    p.vy += (p.baseVy - p.vy) * 0.015

    // 微小随机扰动让运动更自然
    p.vx += (Math.random() - 0.5) * 0.02
    p.vy += (Math.random() - 0.5) * 0.02

    // 偶尔微调方向
    if (Math.random() < 0.003) {
      p.baseVx = (Math.random() - 0.5) * CONFIG.particleSpeed * 2
      p.baseVy = (Math.random() - 0.5) * CONFIG.particleSpeed * 2
    }

    // 鼠标悬浮时微弱吸引
    const dx = mouse.x - p.x
    const dy = mouse.y - p.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    if (dist < CONFIG.hoverRadius && dist > 1) {
      const force = CONFIG.hoverForce * (1 - dist / CONFIG.hoverRadius)
      p.vx += (dx / dist) * force
      p.vy += (dy / dist) * force
    }

    // 颜色缓慢恢复
    const targetHue = randomHue()
    p.hue += (targetHue - p.hue) * 0.001
    if (Math.random() < 0.002) {
      p.color = hueToRgbStr(p.hue)
    }
  }

  // 速度阻尼
  p.vx *= 0.985
  p.vy *= 0.985

  // 限速
  const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy)
  const maxSpeed = mouse.isDown ? 12 : 3
  if (speed > maxSpeed) {
    p.vx = (p.vx / speed) * maxSpeed
    p.vy = (p.vy / speed) * maxSpeed
  }

  p.x += p.vx
  p.y += p.vy

  // 呼吸脉动
  const pulse = Math.sin(time * 0.03 + p.pulseOffset) * 0.3 + 1
  p.size = p.baseSize * pulse

  // 柔和边界（反弹 + 缓冲区）
  const margin = 10
  if (p.x < -margin) { p.x = -margin; p.vx = Math.abs(p.vx) * 0.5; p.baseVx = Math.abs(p.baseVx) }
  if (p.x > w + margin) { p.x = w + margin; p.vx = -Math.abs(p.vx) * 0.5; p.baseVx = -Math.abs(p.baseVx) }
  if (p.y < -margin) { p.y = -margin; p.vy = Math.abs(p.vy) * 0.5; p.baseVy = Math.abs(p.baseVy) }
  if (p.y > h + margin) { p.y = h + margin; p.vy = -Math.abs(p.vy) * 0.5; p.baseVy = -Math.abs(p.baseVy) }
}

// ========== 绘制星空背景 ==========
function drawStars() {
  if (!ctx) return
  for (const s of stars) {
    const twinkle = Math.sin(time * s.twinkleSpeed + s.twinkleOffset)
    const alpha = s.alpha * (0.5 + twinkle * 0.5)
    ctx.beginPath()
    ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(255, 255, 255, ${alpha})`
    ctx.fill()
  }
}

// ========== 绘制粒子拖尾 ==========
function drawTrail(p: Particle) {
  if (!ctx || p.trail.length < 2) return
  for (let i = 1; i < p.trail.length; i++) {
    const t = p.trail[i]
    const prev = p.trail[i - 1]
    const alphaRatio = Math.pow(CONFIG.trailAlphaDecay, i)
    const sizeRatio = 1 - (i / p.trail.length) * 0.6
    const alpha = p.alpha * alphaRatio * 0.4

    ctx.beginPath()
    ctx.moveTo(prev.x, prev.y)
    ctx.lineTo(t.x, t.y)
    ctx.strokeStyle = `rgba(${p.color}, ${alpha})`
    ctx.lineWidth = p.size * sizeRatio * 1.5
    ctx.lineCap = 'round'
    ctx.stroke()
  }
}

// ========== 绘制连线 ==========
function drawConnections() {
  if (!ctx) return
  // 只对中层和近层粒子连线
  const connectable = particles.filter(p => p.layer >= 1)
  for (let i = 0; i < connectable.length; i++) {
    for (let j = i + 1; j < connectable.length; j++) {
      const a = connectable[i]
      const b = connectable[j]
      const dx = a.x - b.x
      const dy = a.y - b.y
      const dist = Math.sqrt(dx * dx + dy * dy)
      if (dist < CONFIG.connectionDistance) {
        const ratio = 1 - dist / CONFIG.connectionDistance
        const alpha = ratio * CONFIG.connectionMaxAlpha

        // 渐变连线
        const grad = ctx.createLinearGradient(a.x, a.y, b.x, b.y)
        grad.addColorStop(0, `rgba(${a.color}, ${alpha})`)
        grad.addColorStop(1, `rgba(${b.color}, ${alpha})`)

        ctx.beginPath()
        ctx.strokeStyle = grad
        ctx.lineWidth = ratio * 1.2
        ctx.moveTo(a.x, a.y)
        ctx.lineTo(b.x, b.y)
        ctx.stroke()
      }
    }
  }
}

// ========== 鼠标光环 ==========
function drawMouseGlow() {
  if (!ctx) return
  if (mouse.x < -999) return

  if (mouse.isDown) {
    // 螺旋时的漩涡光环
    const elapsed = time - mouse.downTime
    const pulseRadius = 80 + Math.sin(elapsed * 0.05) * 20
    const outerRadius = CONFIG.spiralInfluenceRadius * 0.6

    // 外层大光圈
    const grad2 = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, outerRadius)
    const hue1 = (time * 3) % 360
    const hue2 = (time * 3 + 120) % 360
    const [r1, g1, b1] = hslToRgb(hue1, 80, 60)
    const [r2, g2, b2] = hslToRgb(hue2, 80, 60)
    grad2.addColorStop(0, `rgba(${r1}, ${g1}, ${b1}, 0.12)`)
    grad2.addColorStop(0.4, `rgba(${r2}, ${g2}, ${b2}, 0.05)`)
    grad2.addColorStop(1, `rgba(0, 0, 0, 0)`)
    ctx.fillStyle = grad2
    ctx.beginPath()
    ctx.arc(mouse.x, mouse.y, outerRadius, 0, Math.PI * 2)
    ctx.fill()

    // 内层核心光
    const grad1 = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, pulseRadius)
    grad1.addColorStop(0, `rgba(255, 255, 255, 0.15)`)
    grad1.addColorStop(0.5, `rgba(${r1}, ${g1}, ${b1}, 0.08)`)
    grad1.addColorStop(1, `rgba(0, 0, 0, 0)`)
    ctx.fillStyle = grad1
    ctx.beginPath()
    ctx.arc(mouse.x, mouse.y, pulseRadius, 0, Math.PI * 2)
    ctx.fill()

    // 旋转光线
    const lineCount = 6
    for (let i = 0; i < lineCount; i++) {
      const angle = (i / lineCount) * Math.PI * 2 + time * 0.02
      const len = pulseRadius * 1.5
      const endX = mouse.x + Math.cos(angle) * len
      const endY = mouse.y + Math.sin(angle) * len
      const lineGrad = ctx.createLinearGradient(mouse.x, mouse.y, endX, endY)
      lineGrad.addColorStop(0, `rgba(255, 255, 255, 0.08)`)
      lineGrad.addColorStop(1, `rgba(255, 255, 255, 0)`)
      ctx.beginPath()
      ctx.strokeStyle = lineGrad
      ctx.lineWidth = 1.5
      ctx.moveTo(mouse.x, mouse.y)
      ctx.lineTo(endX, endY)
      ctx.stroke()
    }
  } else {
    // 悬浮时的柔和光环
    const grad = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, CONFIG.hoverRadius * 0.6)
    grad.addColorStop(0, `rgba(147, 130, 255, 0.06)`)
    grad.addColorStop(0.5, `rgba(100, 180, 255, 0.03)`)
    grad.addColorStop(1, `rgba(0, 0, 0, 0)`)
    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.arc(mouse.x, mouse.y, CONFIG.hoverRadius * 0.6, 0, Math.PI * 2)
    ctx.fill()
  }
}

// ========== 绘制粒子 ==========
function drawParticle(p: Particle) {
  if (!ctx) return

  // 绘制拖尾
  drawTrail(p)

  // 主体
  ctx.beginPath()
  ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2)
  ctx.fillStyle = `rgba(${p.color}, ${p.alpha})`
  ctx.fill()

  // 内发光
  const glow1 = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.size * 3)
  glow1.addColorStop(0, `rgba(${p.color}, ${p.alpha * 0.3})`)
  glow1.addColorStop(1, `rgba(${p.color}, 0)`)
  ctx.fillStyle = glow1
  ctx.beginPath()
  ctx.arc(p.x, p.y, p.size * 3, 0, Math.PI * 2)
  ctx.fill()

  // 螺旋中时额外的大发光
  if (mouse.isDown) {
    const dx = p.x - mouse.x
    const dy = p.y - mouse.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    if (dist < CONFIG.spiralInfluenceRadius) {
      const intensity = (1 - dist / CONFIG.spiralInfluenceRadius) * 0.2
      const glow2 = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.size * 6)
      glow2.addColorStop(0, `rgba(${p.color}, ${intensity})`)
      glow2.addColorStop(1, `rgba(${p.color}, 0)`)
      ctx.fillStyle = glow2
      ctx.beginPath()
      ctx.arc(p.x, p.y, p.size * 6, 0, Math.PI * 2)
      ctx.fill()
    }
  }
}

// ========== 绘制背景渐变 ==========
function drawBackground() {
  if (!ctx) return
  const w = window.innerWidth
  const h = window.innerHeight

  // 深色渐变背景
  const bg = ctx.createRadialGradient(w * 0.5, h * 0.5, 0, w * 0.5, h * 0.5, Math.max(w, h) * 0.7)
  bg.addColorStop(0, '#0f0f2e')
  bg.addColorStop(0.5, '#0a0a20')
  bg.addColorStop(1, '#050510')
  ctx.fillStyle = bg
  ctx.fillRect(0, 0, w, h)

  // 微妙的星云效果 - 随时间缓慢移动
  const nebulaX = w * 0.3 + Math.sin(time * 0.003) * 100
  const nebulaY = h * 0.4 + Math.cos(time * 0.002) * 80
  const nebula1 = ctx.createRadialGradient(nebulaX, nebulaY, 0, nebulaX, nebulaY, 350)
  nebula1.addColorStop(0, 'rgba(100, 40, 180, 0.03)')
  nebula1.addColorStop(0.5, 'rgba(60, 20, 120, 0.015)')
  nebula1.addColorStop(1, 'rgba(0, 0, 0, 0)')
  ctx.fillStyle = nebula1
  ctx.fillRect(0, 0, w, h)

  const nebula2X = w * 0.7 + Math.cos(time * 0.004) * 120
  const nebula2Y = h * 0.6 + Math.sin(time * 0.003) * 90
  const nebula2 = ctx.createRadialGradient(nebula2X, nebula2Y, 0, nebula2X, nebula2Y, 300)
  nebula2.addColorStop(0, 'rgba(30, 80, 180, 0.025)')
  nebula2.addColorStop(0.5, 'rgba(15, 40, 100, 0.012)')
  nebula2.addColorStop(1, 'rgba(0, 0, 0, 0)')
  ctx.fillStyle = nebula2
  ctx.fillRect(0, 0, w, h)
}

// ========== 主绘制循环 ==========
function draw() {
  if (!ctx) return
  time++

  // 背景
  drawBackground()

  // 星星
  drawStars()

  // 鼠标光环（在粒子下面）
  drawMouseGlow()

  // 更新所有粒子
  for (const p of particles) {
    updateParticle(p)
  }

  // 连线
  drawConnections()

  // 分层绘制粒子（远层先画）
  const layered = [...particles].sort((a, b) => a.layer - b.layer)
  for (const p of layered) {
    drawParticle(p)
  }

  animationId = requestAnimationFrame(draw)
}

// ========== 生命周期 ==========
onMounted(() => {
  resize()
  const w = window.innerWidth
  const h = window.innerHeight

  // 创建背景星星
  for (let i = 0; i < CONFIG.starCount; i++) {
    stars.push(createStar(w, h))
  }

  // 创建多层粒子
  const layerDistribution = [
    { layer: 0, count: Math.floor(CONFIG.particleCount * 0.25) },  // 远景
    { layer: 1, count: Math.floor(CONFIG.particleCount * 0.50) },  // 中景
    { layer: 2, count: Math.floor(CONFIG.particleCount * 0.25) },  // 近景
  ]
  for (const { layer, count } of layerDistribution) {
    for (let i = 0; i < count; i++) {
      particles.push(createParticle(w, h, layer))
    }
  }

  draw()

  window.addEventListener('resize', handleResize)
  window.addEventListener('mousedown', onMouseDown)
  window.addEventListener('mouseup', onMouseUp)
  window.addEventListener('mousemove', onMouseMove)
  window.addEventListener('touchstart', onTouchStart, { passive: true })
  window.addEventListener('touchend', onTouchEnd)
  window.addEventListener('touchmove', onTouchMove, { passive: true })
})

function handleResize() {
  resize()
  // 重新分布星星
  const w = window.innerWidth
  const h = window.innerHeight
  stars = []
  for (let i = 0; i < CONFIG.starCount; i++) {
    stars.push(createStar(w, h))
  }
}

onUnmounted(() => {
  cancelAnimationFrame(animationId)
  window.removeEventListener('resize', handleResize)
  window.removeEventListener('mousedown', onMouseDown)
  window.removeEventListener('mouseup', onMouseUp)
  window.removeEventListener('mousemove', onMouseMove)
  window.removeEventListener('touchstart', onTouchStart)
  window.removeEventListener('touchend', onTouchEnd)
  window.removeEventListener('touchmove', onTouchMove)
})
</script>

<template>
  <canvas ref="canvasRef" class="particle-canvas" />
</template>

<style scoped>
.particle-canvas {
  display: block;
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 0;
}
</style>
