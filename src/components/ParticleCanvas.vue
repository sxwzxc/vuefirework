<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue'

const canvasRef = ref<HTMLCanvasElement | null>(null)
let animationId = 0
let ctx: CanvasRenderingContext2D | null = null
let time = 0
let lastTime = 0

// ========== 配置面板 ==========
const showPanel = ref(true)

const config = reactive({
  // 粒子
  particleCount: 250,
  particleMinSize: 0.8,
  particleMaxSize: 3.5,
  particleSpeed: 1.0,
  particleOpacity: 0.85,
  // 背景
  starCount: 150,
  nebulaIntensity: 0.04,
  backgroundPulse: true,
  // 螺旋
  spiralSpeed: 0.1,
  spiralShrink: 0.986,
  spiralMinDist: 5,
  spiralInfluenceRadius: 450,
  // 连线
  connectionDistance: 140,
  connectionMaxAlpha: 0.22,
  enableConnections: true,
  // 鼠标悬浮吸引
  hoverRadius: 220,
  hoverForce: 0.005,
  // 拖尾
  trailLength: 8,
  trailAlphaDecay: 0.6,
  enableTrails: true,
  // 特效
  enableShootingStars: true,
  shootingStarFrequency: 0.003,
  enableBurstOnClick: true,
  burstParticleCount: 30,
  enableRipple: true,
  enableBrownianMotion: true,
  brownianStrength: 0.04,
  enableColorShift: true,
  colorShiftSpeed: 0.5,
  enableSizeFlicker: true,
  // 物理
  friction: 0.985,
  maxSpeed: 4,
  maxSpeedSpiral: 14,
  bounceEnergy: 0.6,
  // 颜色模式
  colorMode: 'rainbow' as 'rainbow' | 'neon' | 'fire' | 'ocean' | 'matrix' | 'custom',
  customHue: 270,
  // 粒子形状
  particleShape: 'circle' as 'circle' | 'star' | 'diamond' | 'mixed',
  // 发光
  glowIntensity: 0.35,
  enableOuterGlow: true,
})

// ========== 鼠标状态 ==========
let mouse = { x: -9999, y: -9999, isDown: false, downTime: 0, clicks: [] as ClickRipple[] }

interface ClickRipple {
  x: number
  y: number
  radius: number
  maxRadius: number
  alpha: number
  hue: number
  time: number
}

// ========== 接口 ==========
interface Trail { x: number; y: number }

interface Particle {
  x: number; y: number
  vx: number; vy: number
  size: number; baseSize: number
  color: string
  hue: number
  alpha: number; targetAlpha: number
  baseVx: number; baseVy: number
  pulseOffset: number
  trail: Trail[]
  layer: number
  shape: 'circle' | 'star' | 'diamond'
  life: number  // -1 = immortal
  maxLife: number
  isBurst: boolean
  flickerTimer: number
  angle: number  // rotation angle for non-circle shapes
  angularVel: number
  // 螺旋轨迹随机性
  orbitEccentricity: number  // 椭圆偏心率 0-1
  orbitAngleOffset: number   // 轨道角度偏移
  orbitNoise: number         // 轨道噪声
  wasInSpiral: boolean       // 是否刚从螺旋中释放
  releaseTime: number        // 释放时间
}

interface Star {
  x: number; y: number
  size: number; alpha: number
  twinkleSpeed: number; twinkleOffset: number
  color: string
}

interface ShootingStar {
  x: number; y: number
  vx: number; vy: number
  length: number
  alpha: number
  hue: number
  life: number
  maxLife: number
}

let particles: Particle[] = []
let stars: Star[] = []
let shootingStars: ShootingStar[] = []

// ========== 颜色工具 ==========
function hslToRgb(h: number, s: number, l: number): [number, number, number] {
  h = ((h % 360) + 360) % 360
  s /= 100; l /= 100
  const k = (n: number) => (n + h / 30) % 12
  const a = s * Math.min(l, 1 - l)
  const f = (n: number) => l - a * Math.max(-1, Math.min(k(n) - 3, Math.min(9 - k(n), 1)))
  return [Math.round(f(0) * 255), Math.round(f(8) * 255), Math.round(f(4) * 255)]
}

function hueToRgbStr(hue: number, sat = 85, light = 60): string {
  const [r, g, b] = hslToRgb(hue, sat, light)
  return `${r}, ${g}, ${b}`
}

// 根据颜色模式生成色相
function getHueForMode(): number {
  switch (config.colorMode) {
    case 'rainbow': return Math.random() * 360
    case 'neon': return [280, 320, 180, 60, 200][Math.floor(Math.random() * 5)] + (Math.random() - 0.5) * 30
    case 'fire': return Math.random() * 60 - 10  // -10 to 50 (red-orange-yellow)
    case 'ocean': return 180 + Math.random() * 60  // 180-240 (cyan-blue)
    case 'matrix': return 100 + Math.random() * 40  // 100-140 (green)
    case 'custom': return config.customHue + (Math.random() - 0.5) * 40
    default: return Math.random() * 360
  }
}

function getColorSatLight(): [number, number] {
  switch (config.colorMode) {
    case 'neon': return [100, 65]
    case 'fire': return [95, 55]
    case 'ocean': return [80, 55]
    case 'matrix': return [90, 50]
    default: return [85, 60]
  }
}

// ========== 粒子形状 ==========
function getShape(): 'circle' | 'star' | 'diamond' {
  if (config.particleShape === 'mixed') {
    const r = Math.random()
    if (r < 0.5) return 'circle'
    if (r < 0.8) return 'star'
    return 'diamond'
  }
  return config.particleShape
}

// ========== 创建粒子 ==========
function createParticle(w: number, h: number, layer: number = 1, overrides?: Partial<Particle>): Particle {
  const speedMul = layer === 0 ? 0.3 : layer === 1 ? 0.7 : 1.0
  const sizeMul = layer === 0 ? 0.5 : layer === 1 ? 1.0 : 1.4
  const vx = (Math.random() - 0.5) * config.particleSpeed * speedMul * 2
  const vy = (Math.random() - 0.5) * config.particleSpeed * speedMul * 2
  const hue = getHueForMode()
  const [sat, light] = getColorSatLight()
  const baseSize = (Math.random() * (config.particleMaxSize - config.particleMinSize) + config.particleMinSize) * sizeMul
  const baseAlpha = (Math.random() * 0.3 + 0.7) * config.particleOpacity * (layer === 0 ? 0.4 : layer === 1 ? 0.7 : 1.0)

  return {
    x: Math.random() * w,
    y: Math.random() * h,
    vx, vy,
    size: baseSize, baseSize,
    color: hueToRgbStr(hue, sat, light),
    hue,
    alpha: baseAlpha, targetAlpha: baseAlpha,
    baseVx: vx, baseVy: vy,
    pulseOffset: Math.random() * Math.PI * 2,
    trail: [],
    layer,
    shape: getShape(),
    life: -1, maxLife: -1,
    isBurst: false,
    flickerTimer: Math.random() * 100,
    orbitEccentricity: 0.2 + Math.random() * 0.6,
    orbitAngleOffset: Math.random() * Math.PI * 2,
    orbitNoise: 0.3 + Math.random() * 0.7,
    wasInSpiral: false,
    releaseTime: 0,
    angle: Math.random() * Math.PI * 2,
    angularVel: (Math.random() - 0.5) * 0.02,
    ...overrides,
  }
}

function createBurstParticle(x: number, y: number): Particle {
  const angle = Math.random() * Math.PI * 2
  const speed = 2 + Math.random() * 6
  const hue = getHueForMode()
  const [sat, light] = getColorSatLight()
  const life = 40 + Math.random() * 60
  return {
    x, y,
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    size: Math.random() * 2.5 + 0.5,
    baseSize: Math.random() * 2.5 + 0.5,
    color: hueToRgbStr(hue, sat, light),
    hue,
    alpha: 1, targetAlpha: 1,
    baseVx: 0, baseVy: 0,
    pulseOffset: Math.random() * Math.PI * 2,
    trail: [],
    layer: 2,
    shape: getShape(),
    life, maxLife: life,
    isBurst: true,
    flickerTimer: 0,
    orbitEccentricity: 0.3 + Math.random() * 0.5,
    orbitAngleOffset: Math.random() * Math.PI * 2,
    orbitNoise: 0.5 + Math.random() * 0.5,
    wasInSpiral: false,
    releaseTime: 0,
    angle: Math.random() * Math.PI * 2,
    angularVel: (Math.random() - 0.5) * 0.1,
  }
}

function createStar(w: number, h: number): Star {
  const isColored = Math.random() < 0.15
  const hue = isColored ? getHueForMode() : 0
  const color = isColored ? hueToRgbStr(hue, 40, 85) : '255, 255, 255'
  return {
    x: Math.random() * w,
    y: Math.random() * h,
    size: Math.random() * 1.8 + 0.2,
    alpha: Math.random() * 0.6 + 0.15,
    twinkleSpeed: Math.random() * 0.03 + 0.003,
    twinkleOffset: Math.random() * Math.PI * 2,
    color,
  }
}

function createShootingStar(w: number, h: number): ShootingStar {
  const fromLeft = Math.random() < 0.5
  const hue = getHueForMode()
  return {
    x: fromLeft ? -20 : w + 20,
    y: Math.random() * h * 0.6,
    vx: (fromLeft ? 1 : -1) * (8 + Math.random() * 12),
    vy: 2 + Math.random() * 4,
    length: 40 + Math.random() * 80,
    alpha: 0.6 + Math.random() * 0.4,
    hue,
    life: 60 + Math.random() * 80,
    maxLife: 60 + Math.random() * 80,
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
  mouse.x = e.clientX; mouse.y = e.clientY
  mouse.isDown = true; mouse.downTime = time
  // 爆裂效果
  if (config.enableBurstOnClick) {
    for (let i = 0; i < config.burstParticleCount; i++) {
      particles.push(createBurstParticle(e.clientX, e.clientY))
    }
  }
  // 涟漪效果
  if (config.enableRipple) {
    mouse.clicks.push({
      x: e.clientX, y: e.clientY,
      radius: 0, maxRadius: 200 + Math.random() * 100,
      alpha: 0.5, hue: getHueForMode(), time: time,
    })
  }
}
function onMouseUp() { mouse.isDown = false }
function onMouseMove(e: MouseEvent) { mouse.x = e.clientX; mouse.y = e.clientY }
function onTouchStart(e: TouchEvent) {
  const t = e.touches[0]
  mouse.x = t.clientX; mouse.y = t.clientY
  mouse.isDown = true; mouse.downTime = time
  if (config.enableBurstOnClick) {
    for (let i = 0; i < config.burstParticleCount; i++) {
      particles.push(createBurstParticle(t.clientX, t.clientY))
    }
  }
  if (config.enableRipple) {
    mouse.clicks.push({
      x: t.clientX, y: t.clientY,
      radius: 0, maxRadius: 200 + Math.random() * 100,
      alpha: 0.5, hue: getHueForMode(), time: time,
    })
  }
}
function onTouchEnd() { mouse.isDown = false }
function onTouchMove(e: TouchEvent) {
  const t = e.touches[0]; mouse.x = t.clientX; mouse.y = t.clientY
}

// ========== 粒子更新 ==========
function updateParticle(p: Particle, index: number) {
  const w = window.innerWidth
  const h = window.innerHeight

  // 存活检查
  if (p.life > 0) {
    p.life--
    if (p.life <= 0) return false
    // 爆裂粒子渐隐
    if (p.isBurst) {
      p.alpha = (p.life / p.maxLife) * p.targetAlpha
    }
  }

  // 拖尾
  if (config.enableTrails) {
    p.trail.unshift({ x: p.x, y: p.y })
    if (p.trail.length > config.trailLength) p.trail.pop()
  } else {
    p.trail = []
  }

  // 旋转
  p.angle += p.angularVel

  if (mouse.isDown && !p.isBurst) {
    // === 不规则螺旋吸引 ===
    p.wasInSpiral = true
    p.releaseTime = time
    
    const dx = p.x - mouse.x
    const dy = p.y - mouse.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    const angle = Math.atan2(dy, dx)

    const influence = Math.min(1, config.spiralInfluenceRadius / (dist + 1))
    
    // 动态旋转速度：距离越远旋转越慢（飞来时），距离越近旋转越快（螺旋时）
    let rotSpeed: number
    if (dist > config.spiralInfluenceRadius * 0.5) {
      // 外围：较慢的螺旋接近
      rotSpeed = config.spiralSpeed * 0.4 * (1 + 30 / (dist + 10))
    } else {
      // 内圈：快速螺旋
      rotSpeed = config.spiralSpeed * (1 + 100 / (dist + 15))
    }
    
    // 椭圆轨道：使用偏心率让轨道变形
    const spiralAngle = angle + rotSpeed + p.orbitAngleOffset
    let targetDist = Math.max(dist * config.spiralShrink, config.spiralMinDist)
    
    // 椭圆变形：根据角度调整距离（模拟椭圆）
    const ellipseModulation = 1 + p.orbitEccentricity * Math.cos(spiralAngle * 2)
    targetDist *= ellipseModulation
    
    // 添加噪声扰动：让轨迹不规则
    const noiseX = Math.sin(time * 0.05 + p.pulseOffset) * p.orbitNoise * 15
    const noiseY = Math.cos(time * 0.07 + p.pulseOffset * 1.3) * p.orbitNoise * 15
    
    // 随机甩出效果：偶尔给粒子一个向外的冲击
    if (Math.random() < 0.008 && dist < config.spiralInfluenceRadius * 0.7) {
      const flingAngle = angle + (Math.random() - 0.5) * 0.5
      const flingForce = 2 + Math.random() * 3
      p.vx += Math.cos(flingAngle) * flingForce
      p.vy += Math.sin(flingAngle) * flingForce
    }

    const targetX = mouse.x + Math.cos(spiralAngle) * targetDist + noiseX
    const targetY = mouse.y + Math.sin(spiralAngle) * targetDist + noiseY

    // 外围粒子用较弱的吸引力，让螺旋接近更明显
    const attractionStrength = dist > config.spiralInfluenceRadius * 0.5 ? 0.04 : 0.06
    p.vx += (targetX - p.x) * attractionStrength * influence
    p.vy += (targetY - p.y) * attractionStrength * influence

    // 螺旋彩虹变色
    if (config.enableColorShift) {
      p.hue = ((angle / Math.PI) * 180 + time * config.colorShiftSpeed * 4) % 360
      const [sat, light] = getColorSatLight()
      p.color = hueToRgbStr(p.hue, sat, light)
    }

    // 角速度加快
    p.angularVel += (rotSpeed * 0.1 - p.angularVel) * 0.05
  } else if (!p.isBurst) {
    // === 自由漂浮 ===
    
    // 刚释放时快速散开
    if (p.wasInSpiral) {
      const timeSinceRelease = time - p.releaseTime
      if (timeSinceRelease < 30) {
        // 散开力度随时间衰减
        const scatterStrength = (1 - timeSinceRelease / 30) * 0.8
        const dx = p.x - mouse.x
        const dy = p.y - mouse.y
        const dist = Math.sqrt(dx * dx + dy * dy)
        if (dist > 1) {
          // 向外推力
          const pushForce = scatterStrength * 8
          p.vx += (dx / dist) * pushForce
          p.vy += (dy / dist) * pushForce
          
          // 添加随机扰动让散开更自然
          p.vx += (Math.random() - 0.5) * scatterStrength * 4
          p.vy += (Math.random() - 0.5) * scatterStrength * 4
        }
      } else {
        p.wasInSpiral = false
        // 重新随机化基础速度
        p.baseVx = (Math.random() - 0.5) * config.particleSpeed * 2
        p.baseVy = (Math.random() - 0.5) * config.particleSpeed * 2
      }
    }
    
    p.vx += (p.baseVx - p.vx) * 0.012
    p.vy += (p.baseVy - p.vy) * 0.012

    // 布朗运动
    if (config.enableBrownianMotion) {
      p.vx += (Math.random() - 0.5) * config.brownianStrength
      p.vy += (Math.random() - 0.5) * config.brownianStrength
    }

    // 随机方向变化
    if (Math.random() < 0.004) {
      p.baseVx = (Math.random() - 0.5) * config.particleSpeed * 2
      p.baseVy = (Math.random() - 0.5) * config.particleSpeed * 2
    }

    // 随机速度脉冲
    if (Math.random() < 0.001) {
      const burstAngle = Math.random() * Math.PI * 2
      const burstSpeed = 1 + Math.random() * 2
      p.vx += Math.cos(burstAngle) * burstSpeed
      p.vy += Math.sin(burstAngle) * burstSpeed
    }

    // 悬浮吸引
    const dx = mouse.x - p.x
    const dy = mouse.y - p.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    if (dist < config.hoverRadius && dist > 1) {
      const force = config.hoverForce * (1 - dist / config.hoverRadius)
      p.vx += (dx / dist) * force
      p.vy += (dy / dist) * force
    }

    // 颜色缓慢随机漂移
    if (config.enableColorShift && Math.random() < 0.003) {
      p.hue += (Math.random() - 0.5) * config.colorShiftSpeed * 10
      const [sat, light] = getColorSatLight()
      p.color = hueToRgbStr(p.hue, sat, light)
    }

    // 角速度恢复
    p.angularVel += (0.01 * (Math.random() - 0.5) - p.angularVel) * 0.02
  }

  // 涟漪推力
  for (const ripple of mouse.clicks) {
    const dx = p.x - ripple.x
    const dy = p.y - ripple.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    if (Math.abs(dist - ripple.radius) < 30 && dist > 1) {
      const pushForce = ripple.alpha * 0.8
      p.vx += (dx / dist) * pushForce
      p.vy += (dy / dist) * pushForce
    }
  }

  // 阻尼
  p.vx *= config.friction
  p.vy *= config.friction

  // 限速
  const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy)
  const maxSpd = mouse.isDown ? config.maxSpeedSpiral : config.maxSpeed
  if (speed > maxSpd) {
    p.vx = (p.vx / speed) * maxSpd
    p.vy = (p.vy / speed) * maxSpd
  }

  p.x += p.vx
  p.y += p.vy

  // 呼吸脉动
  const pulse = Math.sin(time * 0.025 + p.pulseOffset) * 0.35 + 1
  p.size = p.baseSize * pulse

  // 大小闪烁
  if (config.enableSizeFlicker) {
    p.flickerTimer += 0.05 + Math.random() * 0.02
    if (Math.sin(p.flickerTimer) > 0.95) {
      p.size *= 1.5 + Math.random() * 0.5
    }
  }

  // 边界反弹
  const margin = 15
  if (p.x < -margin) { p.x = -margin; p.vx = Math.abs(p.vx) * config.bounceEnergy; p.baseVx = Math.abs(p.baseVx) }
  if (p.x > w + margin) { p.x = w + margin; p.vx = -Math.abs(p.vx) * config.bounceEnergy; p.baseVx = -Math.abs(p.baseVx) }
  if (p.y < -margin) { p.y = -margin; p.vy = Math.abs(p.vy) * config.bounceEnergy; p.baseVy = Math.abs(p.baseVy) }
  if (p.y > h + margin) { p.y = h + margin; p.vy = -Math.abs(p.vy) * config.bounceEnergy; p.baseVy = -Math.abs(p.baseVy) }

  return true
}

// ========== 绘制背景 ==========
function drawBackground() {
  if (!ctx) return
  const w = window.innerWidth
  const h = window.innerHeight

  // 渐变背景
  const bg = ctx.createRadialGradient(w * 0.5, h * 0.5, 0, w * 0.5, h * 0.5, Math.max(w, h) * 0.75)
  const pulse = config.backgroundPulse ? Math.sin(time * 0.005) * 0.015 : 0
  bg.addColorStop(0, `rgba(15, 15, 48, ${1 + pulse})`)
  bg.addColorStop(0.5, '#0a0a22')
  bg.addColorStop(1, '#040410')
  ctx.fillStyle = bg
  ctx.fillRect(0, 0, w, h)

  // 动态星云
  const intensity = config.nebulaIntensity
  const n1x = w * 0.3 + Math.sin(time * 0.002) * 120
  const n1y = h * 0.35 + Math.cos(time * 0.0015) * 90
  const nHue1 = (time * 0.3) % 360
  const [nr1, ng1, nb1] = hslToRgb(nHue1, 60, 40)
  const neb1 = ctx.createRadialGradient(n1x, n1y, 0, n1x, n1y, 380)
  neb1.addColorStop(0, `rgba(${nr1}, ${ng1}, ${nb1}, ${intensity})`)
  neb1.addColorStop(0.5, `rgba(${nr1}, ${ng1}, ${nb1}, ${intensity * 0.4})`)
  neb1.addColorStop(1, 'rgba(0,0,0,0)')
  ctx.fillStyle = neb1
  ctx.fillRect(0, 0, w, h)

  const n2x = w * 0.72 + Math.cos(time * 0.003) * 140
  const n2y = h * 0.65 + Math.sin(time * 0.0025) * 100
  const nHue2 = (time * 0.3 + 180) % 360
  const [nr2, ng2, nb2] = hslToRgb(nHue2, 50, 35)
  const neb2 = ctx.createRadialGradient(n2x, n2y, 0, n2x, n2y, 320)
  neb2.addColorStop(0, `rgba(${nr2}, ${ng2}, ${nb2}, ${intensity * 0.8})`)
  neb2.addColorStop(0.5, `rgba(${nr2}, ${ng2}, ${nb2}, ${intensity * 0.3})`)
  neb2.addColorStop(1, 'rgba(0,0,0,0)')
  ctx.fillStyle = neb2
  ctx.fillRect(0, 0, w, h)

  // 第三个星云
  const n3x = w * 0.5 + Math.sin(time * 0.001) * 200
  const n3y = h * 0.2 + Math.cos(time * 0.002) * 60
  const nHue3 = (time * 0.2 + 90) % 360
  const [nr3, ng3, nb3] = hslToRgb(nHue3, 45, 30)
  const neb3 = ctx.createRadialGradient(n3x, n3y, 0, n3x, n3y, 250)
  neb3.addColorStop(0, `rgba(${nr3}, ${ng3}, ${nb3}, ${intensity * 0.6})`)
  neb3.addColorStop(1, 'rgba(0,0,0,0)')
  ctx.fillStyle = neb3
  ctx.fillRect(0, 0, w, h)
}

// ========== 绘制星星 ==========
function drawStars() {
  if (!ctx) return
  for (const s of stars) {
    const twinkle = Math.sin(time * s.twinkleSpeed + s.twinkleOffset)
    const alpha = s.alpha * (0.4 + twinkle * 0.6)
    if (alpha <= 0) continue

    ctx.beginPath()
    ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(${s.color}, ${alpha})`
    ctx.fill()

    // 十字光芒效果 (大星星)
    if (s.size > 1.2 && twinkle > 0.3) {
      const sparkAlpha = alpha * 0.4
      const len = s.size * 4 * twinkle
      ctx.strokeStyle = `rgba(${s.color}, ${sparkAlpha})`
      ctx.lineWidth = 0.5
      ctx.beginPath()
      ctx.moveTo(s.x - len, s.y)
      ctx.lineTo(s.x + len, s.y)
      ctx.moveTo(s.x, s.y - len)
      ctx.lineTo(s.x, s.y + len)
      ctx.stroke()
    }
  }
}

// ========== 绘制流星 ==========
function updateAndDrawShootingStars() {
  if (!ctx) return
  const w = window.innerWidth
  const h = window.innerHeight

  // 随机生成新流星
  if (config.enableShootingStars && Math.random() < config.shootingStarFrequency) {
    shootingStars.push(createShootingStar(w, h))
  }

  for (let i = shootingStars.length - 1; i >= 0; i--) {
    const ss = shootingStars[i]
    ss.life--
    if (ss.life <= 0) { shootingStars.splice(i, 1); continue }

    ss.x += ss.vx
    ss.y += ss.vy

    const lifeRatio = ss.life / ss.maxLife
    const alpha = ss.alpha * lifeRatio

    // 流星头
    const [r, g, b] = hslToRgb(ss.hue, 80, 70)
    ctx.beginPath()
    ctx.arc(ss.x, ss.y, 2, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${alpha})`
    ctx.fill()

    // 发光
    const glow = ctx.createRadialGradient(ss.x, ss.y, 0, ss.x, ss.y, 8)
    glow.addColorStop(0, `rgba(255, 255, 255, ${alpha * 0.6})`)
    glow.addColorStop(1, `rgba(${r}, ${g}, ${b}, 0)`)
    ctx.fillStyle = glow
    ctx.beginPath()
    ctx.arc(ss.x, ss.y, 8, 0, Math.PI * 2)
    ctx.fill()

    // 尾巴
    const tailX = ss.x - (ss.vx / Math.sqrt(ss.vx * ss.vx + ss.vy * ss.vy)) * ss.length * lifeRatio
    const tailY = ss.y - (ss.vy / Math.sqrt(ss.vx * ss.vx + ss.vy * ss.vy)) * ss.length * lifeRatio
    const tailGrad = ctx.createLinearGradient(ss.x, ss.y, tailX, tailY)
    tailGrad.addColorStop(0, `rgba(${r}, ${g}, ${b}, ${alpha * 0.8})`)
    tailGrad.addColorStop(0.3, `rgba(${r}, ${g}, ${b}, ${alpha * 0.3})`)
    tailGrad.addColorStop(1, `rgba(${r}, ${g}, ${b}, 0)`)
    ctx.beginPath()
    ctx.strokeStyle = tailGrad
    ctx.lineWidth = 2 * lifeRatio
    ctx.lineCap = 'round'
    ctx.moveTo(ss.x, ss.y)
    ctx.lineTo(tailX, tailY)
    ctx.stroke()

    // 超出边界
    if (ss.x < -100 || ss.x > w + 100 || ss.y > h + 100) {
      shootingStars.splice(i, 1)
    }
  }
}

// ========== 绘制涟漪 ==========
function updateAndDrawRipples() {
  if (!ctx) return
  for (let i = mouse.clicks.length - 1; i >= 0; i--) {
    const r = mouse.clicks[i]
    r.radius += 4
    r.alpha *= 0.96

    if (r.alpha < 0.01 || r.radius > r.maxRadius) {
      mouse.clicks.splice(i, 1)
      continue
    }

    const [cr, cg, cb] = hslToRgb(r.hue, 80, 60)
    ctx.beginPath()
    ctx.arc(r.x, r.y, r.radius, 0, Math.PI * 2)
    ctx.strokeStyle = `rgba(${cr}, ${cg}, ${cb}, ${r.alpha})`
    ctx.lineWidth = 2
    ctx.stroke()

    // 内圈
    if (r.radius > 20) {
      ctx.beginPath()
      ctx.arc(r.x, r.y, r.radius * 0.6, 0, Math.PI * 2)
      ctx.strokeStyle = `rgba(${cr}, ${cg}, ${cb}, ${r.alpha * 0.5})`
      ctx.lineWidth = 1
      ctx.stroke()
    }
  }
}

// ========== 绘制拖尾 ==========
function drawTrail(p: Particle) {
  if (!ctx || p.trail.length < 2 || !config.enableTrails) return
  for (let i = 1; i < p.trail.length; i++) {
    const t = p.trail[i]
    const prev = p.trail[i - 1]
    const alphaRatio = Math.pow(config.trailAlphaDecay, i)
    const sizeRatio = 1 - (i / p.trail.length) * 0.7
    const alpha = p.alpha * alphaRatio * 0.35

    ctx.beginPath()
    ctx.moveTo(prev.x, prev.y)
    ctx.lineTo(t.x, t.y)
    ctx.strokeStyle = `rgba(${p.color}, ${alpha})`
    ctx.lineWidth = p.size * sizeRatio * 1.8
    ctx.lineCap = 'round'
    ctx.stroke()
  }
}

// ========== 绘制连线 ==========
function drawConnections() {
  if (!ctx || !config.enableConnections) return
  const connectable = particles.filter(p => p.layer >= 1 && !p.isBurst)
  const len = connectable.length
  for (let i = 0; i < len; i++) {
    for (let j = i + 1; j < len; j++) {
      const a = connectable[i]
      const b = connectable[j]
      const dx = a.x - b.x
      const dy = a.y - b.y
      const distSq = dx * dx + dy * dy
      const maxDist = config.connectionDistance
      if (distSq < maxDist * maxDist) {
        const dist = Math.sqrt(distSq)
        const ratio = 1 - dist / maxDist
        const alpha = ratio * config.connectionMaxAlpha

        const grad = ctx.createLinearGradient(a.x, a.y, b.x, b.y)
        grad.addColorStop(0, `rgba(${a.color}, ${alpha})`)
        grad.addColorStop(1, `rgba(${b.color}, ${alpha})`)

        ctx.beginPath()
        ctx.strokeStyle = grad
        ctx.lineWidth = ratio * 1.4
        ctx.moveTo(a.x, a.y)
        ctx.lineTo(b.x, b.y)
        ctx.stroke()
      }
    }
  }
}

// ========== 鼠标光环 ==========
function drawMouseGlow() {
  if (!ctx || mouse.x < -999) return

  if (mouse.isDown) {
    const elapsed = time - mouse.downTime
    const pulseR = 90 + Math.sin(elapsed * 0.04) * 25
    const outerR = config.spiralInfluenceRadius * 0.55

    // 外层旋转光环
    const hue1 = (time * 2.5) % 360
    const hue2 = (time * 2.5 + 120) % 360
    const hue3 = (time * 2.5 + 240) % 360
    const [r1, g1, b1] = hslToRgb(hue1, 80, 55)
    const [r2, g2, b2] = hslToRgb(hue2, 80, 55)

    const grad2 = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, outerR)
    grad2.addColorStop(0, `rgba(${r1}, ${g1}, ${b1}, 0.14)`)
    grad2.addColorStop(0.35, `rgba(${r2}, ${g2}, ${b2}, 0.06)`)
    grad2.addColorStop(0.7, `rgba(${r1}, ${g1}, ${b1}, 0.02)`)
    grad2.addColorStop(1, 'rgba(0,0,0,0)')
    ctx.fillStyle = grad2
    ctx.beginPath()
    ctx.arc(mouse.x, mouse.y, outerR, 0, Math.PI * 2)
    ctx.fill()

    // 核心白光
    const grad1 = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, pulseR)
    grad1.addColorStop(0, 'rgba(255,255,255,0.18)')
    grad1.addColorStop(0.4, `rgba(${r1}, ${g1}, ${b1}, 0.1)`)
    grad1.addColorStop(1, 'rgba(0,0,0,0)')
    ctx.fillStyle = grad1
    ctx.beginPath()
    ctx.arc(mouse.x, mouse.y, pulseR, 0, Math.PI * 2)
    ctx.fill()

    // 旋转光线
    const lineCount = 8
    for (let i = 0; i < lineCount; i++) {
      const angle = (i / lineCount) * Math.PI * 2 + time * 0.025
      const len = pulseR * (1.2 + Math.sin(time * 0.03 + i) * 0.5)
      const endX = mouse.x + Math.cos(angle) * len
      const endY = mouse.y + Math.sin(angle) * len
      const [lr, lg, lb] = hslToRgb((hue1 + i * 45) % 360, 70, 60)
      const lineGrad = ctx.createLinearGradient(mouse.x, mouse.y, endX, endY)
      lineGrad.addColorStop(0, `rgba(${lr}, ${lg}, ${lb}, 0.1)`)
      lineGrad.addColorStop(1, `rgba(${lr}, ${lg}, ${lb}, 0)`)
      ctx.beginPath()
      ctx.strokeStyle = lineGrad
      ctx.lineWidth = 1.5
      ctx.moveTo(mouse.x, mouse.y)
      ctx.lineTo(endX, endY)
      ctx.stroke()
    }

    // 螺旋轨道线
    ctx.beginPath()
    const [sr, sg, sb] = hslToRgb(hue3, 70, 55)
    for (let a = 0; a < Math.PI * 6; a += 0.05) {
      const r = 10 + a * 12
      if (r > outerR) break
      const x = mouse.x + Math.cos(a + time * 0.02) * r
      const y = mouse.y + Math.sin(a + time * 0.02) * r
      if (a === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    }
    ctx.strokeStyle = `rgba(${sr}, ${sg}, ${sb}, 0.06)`
    ctx.lineWidth = 1
    ctx.stroke()

  } else {
    // 悬浮柔和光环
    const hoverHue = (time * 0.5) % 360
    const [hr, hg, hb] = hslToRgb(hoverHue, 60, 55)
    const grad = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, config.hoverRadius * 0.55)
    grad.addColorStop(0, `rgba(${hr}, ${hg}, ${hb}, 0.07)`)
    grad.addColorStop(0.5, `rgba(${hr}, ${hg}, ${hb}, 0.03)`)
    grad.addColorStop(1, 'rgba(0,0,0,0)')
    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.arc(mouse.x, mouse.y, config.hoverRadius * 0.55, 0, Math.PI * 2)
    ctx.fill()
  }
}

// ========== 绘制粒子形状 ==========
function drawShape(p: Particle) {
  if (!ctx) return
  ctx.save()
  ctx.translate(p.x, p.y)
  ctx.rotate(p.angle)

  ctx.fillStyle = `rgba(${p.color}, ${p.alpha})`

  switch (p.shape) {
    case 'circle':
      ctx.beginPath()
      ctx.arc(0, 0, p.size, 0, Math.PI * 2)
      ctx.fill()
      break
    case 'star': {
      const spikes = 4
      const outerR = p.size * 1.3
      const innerR = p.size * 0.5
      ctx.beginPath()
      for (let i = 0; i < spikes * 2; i++) {
        const r = i % 2 === 0 ? outerR : innerR
        const angle = (i * Math.PI) / spikes - Math.PI / 2
        const x = Math.cos(angle) * r
        const y = Math.sin(angle) * r
        if (i === 0) ctx.moveTo(x, y)
        else ctx.lineTo(x, y)
      }
      ctx.closePath()
      ctx.fill()
      break
    }
    case 'diamond':
      ctx.beginPath()
      ctx.moveTo(0, -p.size * 1.3)
      ctx.lineTo(p.size * 0.8, 0)
      ctx.lineTo(0, p.size * 1.3)
      ctx.lineTo(-p.size * 0.8, 0)
      ctx.closePath()
      ctx.fill()
      break
  }
  ctx.restore()
}

// ========== 绘制粒子 ==========
function drawParticle(p: Particle) {
  if (!ctx) return

  drawTrail(p)
  drawShape(p)

  // 内发光
  if (config.glowIntensity > 0) {
    const glowR = p.size * 3.5
    const glow1 = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, glowR)
    glow1.addColorStop(0, `rgba(${p.color}, ${p.alpha * config.glowIntensity})`)
    glow1.addColorStop(1, `rgba(${p.color}, 0)`)
    ctx.fillStyle = glow1
    ctx.beginPath()
    ctx.arc(p.x, p.y, glowR, 0, Math.PI * 2)
    ctx.fill()
  }

  // 螺旋中的外发光
  if (config.enableOuterGlow && mouse.isDown) {
    const dx = p.x - mouse.x
    const dy = p.y - mouse.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    if (dist < config.spiralInfluenceRadius) {
      const intensity = (1 - dist / config.spiralInfluenceRadius) * 0.25
      const glowR2 = p.size * 7
      const glow2 = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, glowR2)
      glow2.addColorStop(0, `rgba(${p.color}, ${intensity})`)
      glow2.addColorStop(1, `rgba(${p.color}, 0)`)
      ctx.fillStyle = glow2
      ctx.beginPath()
      ctx.arc(p.x, p.y, glowR2, 0, Math.PI * 2)
      ctx.fill()
    }
  }
}

// ========== FPS ==========
const fps = ref(0)
let frameCount = 0
let fpsTime = 0

// ========== 主绘制循环 ==========
function draw(timestamp: number) {
  if (!ctx) return

  // FPS 计算
  frameCount++
  if (timestamp - fpsTime >= 1000) {
    fps.value = frameCount
    frameCount = 0
    fpsTime = timestamp
  }

  time++

  drawBackground()
  drawStars()
  updateAndDrawShootingStars()
  drawMouseGlow()

  // 更新粒子
  for (let i = particles.length - 1; i >= 0; i--) {
    const alive = updateParticle(particles[i], i)
    if (!alive) particles.splice(i, 1)
  }

  drawConnections()
  updateAndDrawRipples()

  // 分层绘制
  const sorted = [...particles].sort((a, b) => a.layer - b.layer)
  for (const p of sorted) {
    drawParticle(p)
  }

  animationId = requestAnimationFrame(draw)
}

// ========== 重新初始化粒子 ==========
function reinitParticles() {
  const w = window.innerWidth
  const h = window.innerHeight
  particles = []
  const counts = [
    { layer: 0, count: Math.floor(config.particleCount * 0.25) },
    { layer: 1, count: Math.floor(config.particleCount * 0.50) },
    { layer: 2, count: Math.floor(config.particleCount * 0.25) },
  ]
  for (const { layer, count } of counts) {
    for (let i = 0; i < count; i++) {
      particles.push(createParticle(w, h, layer))
    }
  }
}

// 监听粒子数量变化
watch(() => config.particleCount, () => reinitParticles())
watch(() => config.colorMode, () => reinitParticles())
watch(() => config.particleShape, () => reinitParticles())

// ========== 生命周期 ==========
function handleResize() {
  resize()
  const w = window.innerWidth
  const h = window.innerHeight
  stars = []
  for (let i = 0; i < config.starCount; i++) {
    stars.push(createStar(w, h))
  }
}

onMounted(() => {
  resize()
  const w = window.innerWidth
  const h = window.innerHeight

  for (let i = 0; i < config.starCount; i++) {
    stars.push(createStar(w, h))
  }

  reinitParticles()
  animationId = requestAnimationFrame(draw)

  window.addEventListener('resize', handleResize)
  window.addEventListener('mousedown', onMouseDown)
  window.addEventListener('mouseup', onMouseUp)
  window.addEventListener('mousemove', onMouseMove)
  window.addEventListener('touchstart', onTouchStart, { passive: true })
  window.addEventListener('touchend', onTouchEnd)
  window.addEventListener('touchmove', onTouchMove, { passive: true })
})

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

// ========== 预设 ==========
function applyPreset(preset: string) {
  switch (preset) {
    case 'galaxy':
      config.particleCount = 300; config.colorMode = 'rainbow'; config.particleShape = 'mixed'
      config.enableShootingStars = true; config.spiralSpeed = 0.12; config.nebulaIntensity = 0.05
      config.glowIntensity = 0.4; config.trailLength = 8; config.enableBurstOnClick = true
      config.brownianStrength = 0.04; config.connectionDistance = 120
      break
    case 'firefly':
      config.particleCount = 180; config.colorMode = 'fire'; config.particleShape = 'circle'
      config.enableShootingStars = false; config.spiralSpeed = 0.06; config.nebulaIntensity = 0.02
      config.glowIntensity = 0.5; config.trailLength = 12; config.enableBurstOnClick = true
      config.brownianStrength = 0.08; config.connectionDistance = 80
      config.particleMaxSize = 2.5; config.enableSizeFlicker = true
      break
    case 'ocean':
      config.particleCount = 250; config.colorMode = 'ocean'; config.particleShape = 'diamond'
      config.enableShootingStars = false; config.spiralSpeed = 0.05; config.nebulaIntensity = 0.04
      config.glowIntensity = 0.3; config.trailLength = 10; config.enableBurstOnClick = true
      config.brownianStrength = 0.03; config.connectionDistance = 150; config.particleSpeed = 0.6
      break
    case 'matrix':
      config.particleCount = 350; config.colorMode = 'matrix'; config.particleShape = 'diamond'
      config.enableShootingStars = true; config.shootingStarFrequency = 0.008
      config.spiralSpeed = 0.15; config.nebulaIntensity = 0.02
      config.glowIntensity = 0.25; config.trailLength = 6; config.enableBurstOnClick = true
      config.brownianStrength = 0.02; config.connectionDistance = 100
      config.particleSpeed = 1.2; config.enableSizeFlicker = false
      break
    case 'neon':
      config.particleCount = 200; config.colorMode = 'neon'; config.particleShape = 'star'
      config.enableShootingStars = true; config.spiralSpeed = 0.1; config.nebulaIntensity = 0.06
      config.glowIntensity = 0.55; config.trailLength = 10; config.enableBurstOnClick = true
      config.brownianStrength = 0.05; config.connectionDistance = 140
      config.particleMaxSize = 4; config.enableSizeFlicker = true
      break
  }
  reinitParticles()
}
</script>

<template>
  <canvas ref="canvasRef" class="particle-canvas" />

  <!-- 控制面板切换按钮 -->
  <button class="panel-toggle" @click="showPanel = !showPanel" :title="showPanel ? '隐藏面板' : '显示面板'">
    {{ showPanel ? '✕' : '⚙' }}
  </button>

  <!-- 控制面板 -->
  <transition name="panel">
    <div v-if="showPanel" class="control-panel" @mousedown.stop @touchstart.stop>
      <div class="panel-header">
        <span>⚙ 粒子控制面板</span>
        <span class="fps">{{ fps }} FPS</span>
      </div>

      <!-- 预设 -->
      <div class="section">
        <div class="section-title">🎨 预设主题</div>
        <div class="preset-row">
          <button class="preset-btn galaxy" @click="applyPreset('galaxy')">银河</button>
          <button class="preset-btn firefly" @click="applyPreset('firefly')">萤火</button>
          <button class="preset-btn ocean" @click="applyPreset('ocean')">海洋</button>
          <button class="preset-btn matrix" @click="applyPreset('matrix')">矩阵</button>
          <button class="preset-btn neon" @click="applyPreset('neon')">霓虹</button>
        </div>
      </div>

      <!-- 粒子设置 -->
      <div class="section">
        <div class="section-title">✨ 粒子</div>
        <div class="control-row">
          <label>数量: {{ config.particleCount }}</label>
          <input type="range" v-model.number="config.particleCount" min="50" max="600" step="10" />
        </div>
        <div class="control-row">
          <label>最大尺寸: {{ config.particleMaxSize.toFixed(1) }}</label>
          <input type="range" v-model.number="config.particleMaxSize" min="1" max="8" step="0.1" />
        </div>
        <div class="control-row">
          <label>速度: {{ config.particleSpeed.toFixed(1) }}</label>
          <input type="range" v-model.number="config.particleSpeed" min="0.1" max="3" step="0.1" />
        </div>
        <div class="control-row">
          <label>透明度: {{ config.particleOpacity.toFixed(2) }}</label>
          <input type="range" v-model.number="config.particleOpacity" min="0.1" max="1" step="0.05" />
        </div>
        <div class="control-row">
          <label>形状</label>
          <select v-model="config.particleShape">
            <option value="circle">圆形</option>
            <option value="star">星形</option>
            <option value="diamond">菱形</option>
            <option value="mixed">混合</option>
          </select>
        </div>
      </div>

      <!-- 颜色 -->
      <div class="section">
        <div class="section-title">🌈 颜色</div>
        <div class="control-row">
          <label>颜色模式</label>
          <select v-model="config.colorMode">
            <option value="rainbow">彩虹</option>
            <option value="neon">霓虹</option>
            <option value="fire">火焰</option>
            <option value="ocean">海洋</option>
            <option value="matrix">矩阵</option>
            <option value="custom">自定义</option>
          </select>
        </div>
        <div class="control-row" v-if="config.colorMode === 'custom'">
          <label>色相: {{ config.customHue }}°</label>
          <input type="range" v-model.number="config.customHue" min="0" max="360" step="1" />
        </div>
        <div class="control-row">
          <label>颜色漂移速度: {{ config.colorShiftSpeed.toFixed(1) }}</label>
          <input type="range" v-model.number="config.colorShiftSpeed" min="0" max="3" step="0.1" />
        </div>
        <div class="control-row toggle-row">
          <label>颜色变化</label>
          <input type="checkbox" v-model="config.enableColorShift" />
        </div>
      </div>

      <!-- 螺旋 -->
      <div class="section">
        <div class="section-title">🌀 螺旋</div>
        <div class="control-row">
          <label>旋转速度: {{ config.spiralSpeed.toFixed(2) }}</label>
          <input type="range" v-model.number="config.spiralSpeed" min="0.01" max="0.3" step="0.01" />
        </div>
        <div class="control-row">
          <label>收缩率: {{ config.spiralShrink.toFixed(3) }}</label>
          <input type="range" v-model.number="config.spiralShrink" min="0.95" max="0.999" step="0.001" />
        </div>
        <div class="control-row">
          <label>影响范围: {{ config.spiralInfluenceRadius }}</label>
          <input type="range" v-model.number="config.spiralInfluenceRadius" min="100" max="800" step="10" />
        </div>
      </div>

      <!-- 特效 -->
      <div class="section">
        <div class="section-title">💫 特效</div>
        <div class="control-row toggle-row">
          <label>粒子连线</label>
          <input type="checkbox" v-model="config.enableConnections" />
        </div>
        <div class="control-row toggle-row">
          <label>粒子拖尾</label>
          <input type="checkbox" v-model="config.enableTrails" />
        </div>
        <div class="control-row toggle-row">
          <label>流星</label>
          <input type="checkbox" v-model="config.enableShootingStars" />
        </div>
        <div class="control-row toggle-row">
          <label>点击爆裂</label>
          <input type="checkbox" v-model="config.enableBurstOnClick" />
        </div>
        <div class="control-row toggle-row">
          <label>点击涟漪</label>
          <input type="checkbox" v-model="config.enableRipple" />
        </div>
        <div class="control-row toggle-row">
          <label>布朗运动</label>
          <input type="checkbox" v-model="config.enableBrownianMotion" />
        </div>
        <div class="control-row toggle-row">
          <label>大小闪烁</label>
          <input type="checkbox" v-model="config.enableSizeFlicker" />
        </div>
        <div class="control-row">
          <label>发光强度: {{ config.glowIntensity.toFixed(2) }}</label>
          <input type="range" v-model.number="config.glowIntensity" min="0" max="1" step="0.05" />
        </div>
        <div class="control-row">
          <label>拖尾长度: {{ config.trailLength }}</label>
          <input type="range" v-model.number="config.trailLength" min="2" max="20" step="1" />
        </div>
        <div class="control-row">
          <label>连线距离: {{ config.connectionDistance }}</label>
          <input type="range" v-model.number="config.connectionDistance" min="50" max="250" step="5" />
        </div>
        <div class="control-row">
          <label>布朗强度: {{ config.brownianStrength.toFixed(2) }}</label>
          <input type="range" v-model.number="config.brownianStrength" min="0" max="0.2" step="0.01" />
        </div>
        <div class="control-row">
          <label>星云强度: {{ config.nebulaIntensity.toFixed(2) }}</label>
          <input type="range" v-model.number="config.nebulaIntensity" min="0" max="0.15" step="0.005" />
        </div>
      </div>

      <!-- 物理 -->
      <div class="section">
        <div class="section-title">⚡ 物理</div>
        <div class="control-row">
          <label>摩擦力: {{ config.friction.toFixed(3) }}</label>
          <input type="range" v-model.number="config.friction" min="0.95" max="0.999" step="0.001" />
        </div>
        <div class="control-row">
          <label>悬浮吸力: {{ config.hoverForce.toFixed(3) }}</label>
          <input type="range" v-model.number="config.hoverForce" min="0" max="0.02" step="0.001" />
        </div>
        <div class="control-row">
          <label>悬浮范围: {{ config.hoverRadius }}</label>
          <input type="range" v-model.number="config.hoverRadius" min="50" max="400" step="10" />
        </div>
        <div class="control-row">
          <label>反弹能量: {{ config.bounceEnergy.toFixed(1) }}</label>
          <input type="range" v-model.number="config.bounceEnergy" min="0.1" max="1" step="0.1" />
        </div>
      </div>
    </div>
  </transition>
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

/* 面板切换按钮 */
.panel-toggle {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 1001;
  width: 36px;
  height: 36px;
  border: 1px solid rgba(255,255,255,0.15);
  border-radius: 8px;
  background: rgba(10, 10, 30, 0.7);
  color: rgba(255,255,255,0.7);
  font-size: 16px;
  cursor: pointer;
  backdrop-filter: blur(10px);
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}
.panel-toggle:hover {
  background: rgba(20, 20, 50, 0.9);
  color: #fff;
  border-color: rgba(147, 51, 234, 0.5);
}

/* 控制面板 */
.control-panel {
  position: fixed;
  top: 3.5rem;
  right: 1rem;
  z-index: 1000;
  width: 300px;
  max-height: calc(100vh - 5rem);
  overflow-y: auto;
  background: rgba(8, 8, 25, 0.88);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 12px;
  padding: 0;
  backdrop-filter: blur(20px);
  color: rgba(255,255,255,0.85);
  font-size: 12px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.5);
}

.control-panel::-webkit-scrollbar {
  width: 4px;
}
.control-panel::-webkit-scrollbar-track {
  background: transparent;
}
.control-panel::-webkit-scrollbar-thumb {
  background: rgba(255,255,255,0.1);
  border-radius: 2px;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 14px;
  border-bottom: 1px solid rgba(255,255,255,0.06);
  font-size: 13px;
  font-weight: 600;
  letter-spacing: 0.5px;
  position: sticky;
  top: 0;
  background: rgba(8, 8, 25, 0.95);
  border-radius: 12px 12px 0 0;
  z-index: 1;
}

.fps {
  font-size: 11px;
  color: rgba(100, 255, 150, 0.7);
  font-weight: 400;
}

.section {
  padding: 10px 14px;
  border-bottom: 1px solid rgba(255,255,255,0.04);
}

.section-title {
  font-size: 12px;
  font-weight: 600;
  margin-bottom: 8px;
  color: rgba(255,255,255,0.6);
  letter-spacing: 0.5px;
}

.control-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
  gap: 8px;
}

.control-row label {
  flex-shrink: 0;
  font-size: 11px;
  color: rgba(255,255,255,0.55);
  min-width: 80px;
}

.control-row input[type="range"] {
  flex: 1;
  height: 4px;
  -webkit-appearance: none;
  appearance: none;
  background: rgba(255,255,255,0.1);
  border-radius: 2px;
  outline: none;
  cursor: pointer;
}

.control-row input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: rgba(147, 100, 255, 0.9);
  cursor: pointer;
  border: none;
  box-shadow: 0 0 6px rgba(147, 100, 255, 0.4);
}

.control-row select {
  flex: 1;
  background: rgba(255,255,255,0.08);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 4px;
  color: rgba(255,255,255,0.8);
  padding: 3px 6px;
  font-size: 11px;
  cursor: pointer;
  outline: none;
}

.control-row select:focus {
  border-color: rgba(147, 100, 255, 0.5);
}

.toggle-row input[type="checkbox"] {
  -webkit-appearance: none;
  appearance: none;
  width: 32px;
  height: 16px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
  position: relative;
  cursor: pointer;
  transition: background 0.2s;
  flex-shrink: 0;
}

.toggle-row input[type="checkbox"]::after {
  content: '';
  position: absolute;
  top: 2px;
  left: 2px;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: rgba(255,255,255,0.4);
  transition: all 0.2s;
}

.toggle-row input[type="checkbox"]:checked {
  background: rgba(147, 100, 255, 0.6);
}

.toggle-row input[type="checkbox"]:checked::after {
  left: 18px;
  background: #fff;
}

/* 预设按钮 */
.preset-row {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.preset-btn {
  flex: 1;
  min-width: 48px;
  padding: 5px 8px;
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 6px;
  background: rgba(255,255,255,0.05);
  color: rgba(255,255,255,0.7);
  font-size: 11px;
  cursor: pointer;
  transition: all 0.2s;
}

.preset-btn:hover {
  background: rgba(255,255,255,0.12);
  color: #fff;
}

.preset-btn.galaxy { border-color: rgba(147, 51, 234, 0.3); }
.preset-btn.galaxy:hover { background: rgba(147, 51, 234, 0.2); border-color: rgba(147, 51, 234, 0.6); }
.preset-btn.firefly { border-color: rgba(245, 158, 11, 0.3); }
.preset-btn.firefly:hover { background: rgba(245, 158, 11, 0.2); border-color: rgba(245, 158, 11, 0.6); }
.preset-btn.ocean { border-color: rgba(6, 182, 212, 0.3); }
.preset-btn.ocean:hover { background: rgba(6, 182, 212, 0.2); border-color: rgba(6, 182, 212, 0.6); }
.preset-btn.matrix { border-color: rgba(34, 197, 94, 0.3); }
.preset-btn.matrix:hover { background: rgba(34, 197, 94, 0.2); border-color: rgba(34, 197, 94, 0.6); }
.preset-btn.neon { border-color: rgba(236, 72, 153, 0.3); }
.preset-btn.neon:hover { background: rgba(236, 72, 153, 0.2); border-color: rgba(236, 72, 153, 0.6); }

/* 面板过渡动画 */
.panel-enter-active, .panel-leave-active {
  transition: all 0.3s ease;
}
.panel-enter-from, .panel-leave-to {
  opacity: 0;
  transform: translateX(20px);
}
</style>
