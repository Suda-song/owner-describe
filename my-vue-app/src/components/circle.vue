<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, defineComponent, watch } from 'vue'
import { gsap } from 'gsap'
defineComponent({
  name: 'Circle'
})

// 定义props接收外部进度
const props = defineProps<{
  progress: number // 0-100 的数值
}>()
const currentProgress = ref(props.progress)
const canvasRef = ref<HTMLCanvasElement>()
let animationId: number
let isShrinking = ref(false)
let currentRadius = ref(100)
let currentWidth = ref(1)
const innerRadiusRatio = 0.95
let gapMove = ref(0)

const init = (canvas: HTMLCanvasElement) => {
  const dpr = window.devicePixelRatio || 1
  canvas.width = window.innerWidth * dpr
  canvas.height = window.innerHeight * dpr
  canvas.style.width = `${window.innerWidth}px`
  canvas.style.height = `${window.innerHeight}px`

  const ctx = canvas.getContext('2d')
  if (ctx) {
    ctx.scale(dpr, dpr) // 缩放绘图上下文以匹配像素比
  }
}

const draw = (
  ctx: CanvasRenderingContext2D,
  head: number,
  tail: number,
  isComplete: boolean
) => {
  const centerX = window.innerWidth / 2
  const centerY = window.innerHeight / 2
  const radius = currentRadius.value
  const innerRadius = radius * innerRadiusRatio

  ctx.clearRect(0, 0, window.innerWidth, window.innerHeight)
  // draw 内圈
  // ctx.beginPath()
  // ctx.arc(centerX, centerY, radius, 0, Math.PI * 2)

  ctx.beginPath()
  ctx.arc(centerX, centerY, innerRadius, 0, Math.PI * 2)
  const innerGradient = ctx.createLinearGradient(
    centerX - innerRadius,
    centerY - innerRadius,
    centerX + innerRadius,
    centerY + innerRadius
  )
  innerGradient.addColorStop(0, 'rgba(79, 172, 254, 0.3)')
  innerGradient.addColorStop(1, 'rgba(0, 242, 254, 0.3)')
  ctx.lineWidth = 1
  ctx.strokeStyle = innerGradient
  ctx.stroke()

  ctx.beginPath()
  if (isComplete) {
    ctx.arc(centerX, centerY, radius, 0, Math.PI * 2)
  } else {
    const normalizedHead = head % (Math.PI * 2)
    const normalizedTail = tail % (Math.PI * 2)
    ctx.arc(centerX, centerY, radius, normalizedTail, normalizedHead)
  }

  const gradient = ctx.createLinearGradient(
    centerX - radius,
    centerY - radius,
    centerX + radius,
    centerY + radius
  )
  gradient.addColorStop(0, '#4facfe')
  gradient.addColorStop(1, '#00f2fe')

  ctx.strokeStyle = gradient
  ctx.lineWidth = currentWidth.value
  ctx.lineCap = 'round'
  // ctx.shadowBlur = 10
  // ctx.shadowColor = '#00f2fe'
  ctx.stroke()
  // draw gaps
  if (currentProgress.value > 87) {
    const opacity = (currentProgress.value - 87) / 13
    ctx.beginPath()
    ctx.arc(
      centerX,
      centerY,
      radius,
      Math.PI * 1.5 - Math.PI / 36 + gapMove.value,
      Math.PI * 1.5 + Math.PI / 36 + gapMove.value
    )
    ctx.lineWidth = currentWidth.value
    ctx.strokeStyle = `rgba(0, 0, 0, ${opacity})`
    ctx.stroke()
    ctx.beginPath()
    ctx.arc(
      centerX,
      centerY,
      radius,
      Math.PI * 0.5 - Math.PI / 36 + gapMove.value,
      Math.PI * 0.5 + Math.PI / 36 + gapMove.value
    )
    ctx.lineWidth = currentWidth.value + 0.2
    ctx.strokeStyle = `rgba(0, 0, 0, ${opacity})`
    ctx.stroke()
  }
}

let lastProgress = 0
let head = Math.PI * 0.5
let tail = 0
let baseSpeed = Math.PI * 0.5 * 0.01
const animate = (ctx: CanvasRenderingContext2D) => {
  const progress = currentProgress.value / 100

  const phase1 = 0.25

  const deltaProgress = progress - lastProgress
  lastProgress = progress
  if (progress >= 1 && !isShrinking.value) {
    isShrinking.value = true
    startShrinkAnimation()
  }
  if (progress <= phase1) {
    // 第一阶段：匀速运动
    head += baseSpeed
    tail += baseSpeed
  } else {
    if (deltaProgress === 0) {
      head += baseSpeed
      tail += baseSpeed
    } else {
      const t = progress - phase1
      const smoothFactor = 1 / (1 + Math.exp(-t * 8 + 4))
      const decayFactor = smoothFactor * 1.2
      head += baseSpeed + 2 * Math.PI * deltaProgress * (decayFactor + 1)
      tail += baseSpeed + 2 * Math.PI * deltaProgress * decayFactor
    }
  }
  draw(ctx, head, tail, progress >= 1)
  animationId = requestAnimationFrame(() => animate(ctx))
}
const startShrinkAnimation = () => {
  gsap.to(currentRadius, {
    value: 60,
    duration: 1.5,
    ease: 'expo.in',
    onComplete: () => {
      console.log('缩小动画完成')
    }
  })
  gsap.to(currentWidth, {
    value: 0.5,
    duration: 1.5,
    ease: 'expo.in',
    onComplete: () => {
      console.log('缩小动画完成')
    }
  })
  gsap.to(gapMove, {
    value: Math.PI / 3,
    duration: 1.5,
    ease: 'expo.in',
    onComplete: () => {
      console.log('缩小动画完成')
    }
  })
}
// 监听外部进度变化
watch(
  () => props.progress,
  newProgress => {
    // 确保进度在0-100之间
    currentProgress.value = Math.max(0, Math.min(100, newProgress))
  },
  { immediate: true }
)

const handleResize = () => {
  if (!canvasRef.value) return
  const canvas = canvasRef.value
  const oldWidth = canvas.width
  const oldHeight = canvas.height

  // 保存当前画布内容
  const tempCanvas = document.createElement('canvas')
  tempCanvas.width = oldWidth
  tempCanvas.height = oldHeight
  const tempCtx = tempCanvas.getContext('2d')
  if (tempCtx) {
    tempCtx.drawImage(canvas, 0, 0)
  }

  // 调整画布大小
  canvas.width = window.innerWidth
  canvas.height = window.innerHeight
  // 恢复画布内容
  const ctx = canvas.getContext('2d')
  if (ctx && tempCtx) {
    ctx.drawImage(tempCanvas, 0, 0)
  }
}

onMounted(() => {
  const canvas = canvasRef.value

  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.fillStyle = '#000000'
  ctx.fillRect(0, 0, window.innerWidth, window.innerHeight)

  init(canvas)
  animationId = requestAnimationFrame(() => animate(ctx))
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  window.removeEventListener('resize', handleResize)
})
</script>

<template>
  <canvas ref="canvasRef" class="circle"></canvas>
</template>

<style scoped>
body {
  margin: 0;
  padding: 0;
}
html,
body {
  overflow: hidden;
}
canvas {
  position: fixed;
  top: 0;
  left: 0;
  outline: none;
  /* background-color: #000; */
}
</style>
