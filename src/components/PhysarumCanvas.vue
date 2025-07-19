<template>
  <div class="physarum-container">
    <canvas
      ref="canvas"
      class="physarum-canvas"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
    ></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'

const canvas = ref(null)
const props = defineProps({
  numParticles: { type: Number, default: 5000 },
  sensorDistance: { type: Number, default: 9.0 },
  sensorAngle: { type: Number, default: 0.5 },
  rotationAngle: { type: Number, default: 0.3 },
  moveDistance: { type: Number, default: 1.0 },
  decayFactor: { type: Number, default: 0.99 },
  depositAmount: { type: Number, default: 5.0 },
  isPlaying: { type: Boolean, default: true }
})

let ctx = null
let animationId = null
let width = 0
let height = 0
let particles = []
let trailMap = null
let imageData = null
let isMouseDown = false
let mousePos = { x: 0, y: 0 }

const emit = defineEmits(['ready'])

class Particle {
  constructor() {
    this.x = width / 2 + (Math.random() - 0.5) * 200
    this.y = height / 2 + (Math.random() - 0.5) * 200
    this.angle = Math.random() * Math.PI * 2
  }

  update() {
    // Sensing
    const sd = props.sensorDistance
    const sa = props.sensorAngle

    const fx = this.x + Math.cos(this.angle) * sd
    const fy = this.y + Math.sin(this.angle) * sd
    const lx = this.x + Math.cos(this.angle + sa) * sd
    const ly = this.y + Math.sin(this.angle + sa) * sd
    const rx = this.x + Math.cos(this.angle - sa) * sd
    const ry = this.y + Math.sin(this.angle - sa) * sd

    const forwardSense = this.sampleTrail(fx, fy)
    const leftSense = this.sampleTrail(lx, ly)
    const rightSense = this.sampleTrail(rx, ry)

    // Steering
    if (forwardSense > leftSense && forwardSense > rightSense) {
      // Continue forward
    } else if (forwardSense < leftSense && forwardSense < rightSense) {
      // Random turn
      this.angle += (Math.random() - 0.5) * props.rotationAngle * 2
    } else if (leftSense > rightSense) {
      this.angle += props.rotationAngle
    } else {
      this.angle -= props.rotationAngle
    }

    // Mouse attraction
    if (isMouseDown) {
      const dx = mousePos.x - this.x
      const dy = mousePos.y - this.y
      const dist = Math.sqrt(dx * dx + dy * dy)
      if (dist < 100 && dist > 0) {
        const mouseAngle = Math.atan2(dy, dx)
        this.angle = this.angle * 0.9 + mouseAngle * 0.1
      }
    }

    // Move
    this.x += Math.cos(this.angle) * props.moveDistance
    this.y += Math.sin(this.angle) * props.moveDistance

    // Wrap boundaries
    this.x = (this.x + width) % width
    this.y = (this.y + height) % height

    // Deposit trail
    this.depositTrail()
  }

  sampleTrail(x, y) {
    const ix = Math.floor(x + width) % width
    const iy = Math.floor(y + height) % height
    return trailMap[iy * width + ix] || 0
  }

  depositTrail() {
    const ix = Math.floor(this.x)
    const iy = Math.floor(this.y)
    if (ix >= 0 && ix < width && iy >= 0 && iy < height) {
      trailMap[iy * width + ix] = Math.min(trailMap[iy * width + ix] + props.depositAmount, 255)
    }
  }
}

function initializeSimulation() {
  width = canvas.value.width
  height = canvas.value.height

  // Initialize trail map
  trailMap = new Float32Array(width * height)
  trailMap.fill(0)

  // Initialize particles
  particles = []
  for (let i = 0; i < props.numParticles; i++) {
    particles.push(new Particle())
  }

  // Create image data for rendering
  imageData = ctx.createImageData(width, height)
}

function diffuseTrails() {
  const newTrailMap = new Float32Array(width * height)

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      let sum = 0
      let count = 0

      // 3x3 kernel
      for (let dy = -1; dy <= 1; dy++) {
        for (let dx = -1; dx <= 1; dx++) {
          const nx = (x + dx + width) % width
          const ny = (y + dy + height) % height
          sum += trailMap[ny * width + nx]
          count++
        }
      }

      const idx = y * width + x
      newTrailMap[idx] = (sum / count) * props.decayFactor
    }
  }

  trailMap = newTrailMap
}

function render() {
  if (!props.isPlaying) {
    animationId = requestAnimationFrame(render)
    return
  }

  // Update particles
  particles.forEach(particle => particle.update())

  // Diffuse and decay trails
  diffuseTrails()

  // Render to canvas
  for (let i = 0; i < width * height; i++) {
    const intensity = Math.min(trailMap[i] / 255, 1)

    // Create colorful gradient
    let r, g, b
    if (intensity > 0.8) {
      r = 255
      g = 255 * (1 - (intensity - 0.8) * 5)
      b = 0
    } else if (intensity > 0.5) {
      r = 255 * (intensity - 0.5) * 3.33
      g = 255
      b = 255 * (1 - (intensity - 0.5) * 3.33)
    } else if (intensity > 0.2) {
      r = 0
      g = 255 * (intensity - 0.2) * 3.33
      b = 255
    } else {
      r = 0
      g = 0
      b = Math.max(25, 255 * intensity * 5)
    }

    const pixelIndex = i * 4
    imageData.data[pixelIndex] = r
    imageData.data[pixelIndex + 1] = g
    imageData.data[pixelIndex + 2] = b
    imageData.data[pixelIndex + 3] = 255
  }

  ctx.putImageData(imageData, 0, 0)

  animationId = requestAnimationFrame(render)
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  width = Math.floor(rect.width)
  height = Math.floor(rect.height)

  canvas.value.width = width
  canvas.value.height = height

  if (ctx) {
    initializeSimulation()
  }
}

function resetSimulation() {
  if (trailMap) {
    trailMap.fill(0)
    particles.forEach(particle => {
      particle.x = width / 2 + (Math.random() - 0.5) * 200
      particle.y = height / 2 + (Math.random() - 0.5) * 200
      particle.angle = Math.random() * Math.PI * 2
    })
  }
}

function onMouseDown(event) {
  isMouseDown = true
  updateMousePos(event)
}

function onMouseMove(event) {
  updateMousePos(event)
}

function onMouseUp() {
  isMouseDown = false
}

function updateMousePos(event) {
  const rect = canvas.value.getBoundingClientRect()
  mousePos.x = event.clientX - rect.left
  mousePos.y = event.clientY - rect.top
}

watch(() => props.isPlaying, (isPlaying) => {
  if (isPlaying && !animationId) {
    animationId = requestAnimationFrame(render)
  }
})

onMounted(() => {
  ctx = canvas.value.getContext('2d')
  resizeCanvas()
  initializeSimulation()
  emit('ready')
  animationId = requestAnimationFrame(render)

  window.addEventListener('resize', resizeCanvas)
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  window.removeEventListener('resize', resizeCanvas)
})

defineExpose({
  resetSimulation
})
</script>

<style scoped>
.physarum-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  z-index: 0;
}

.physarum-canvas {
  width: 100%;
  height: 100%;
  display: block;
  background: #000;
  cursor: crosshair;
  image-rendering: -moz-crisp-edges;
  image-rendering: -webkit-crisp-edges;
  image-rendering: pixelated;
  image-rendering: crisp-edges;
}
</style>
