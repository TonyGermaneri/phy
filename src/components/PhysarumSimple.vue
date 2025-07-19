<template>
  <div class="physarum-container">
    <canvas
      ref="canvas"
      class="physarum-canvas"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
    ></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'

const canvas = ref(null)
const props = defineProps({
  numParticles: { type: Number, default: 100000 },
  sensorDistance: { type: Number, default: 9.0 },
  sensorAngle: { type: Number, default: 0.5 },
  rotationAngle: { type: Number, default: 0.3 },
  moveDistance: { type: Number, default: 1.0 },
  decayFactor: { type: Number, default: 0.9 },
  depositAmount: { type: Number, default: 5.0 },
  isPlaying: { type: Boolean, default: true }
})

let gl = null
let animationId = null
let program = null
let trailProgram = null
let displayProgram = null
let textures = {}
let buffers = {}
let frameBuffers = {}
let width = 0
let height = 0
let particles = []
let isMouseDown = false
let mousePos = { x: 0, y: 0 }
let time = 0

const emit = defineEmits(['ready'])

// Simple vertex shader
const vertexShaderSource = `
attribute vec2 a_position;
varying vec2 v_texCoord;

void main() {
  gl_Position = vec4(a_position, 0.0, 1.0);
  v_texCoord = (a_position + 1.0) * 0.5;
}
`

// Fragment shader for trail processing
const trailFragmentShader = `
precision mediump float;
uniform sampler2D u_texture;
uniform vec2 u_resolution;
uniform float u_decay;
varying vec2 v_texCoord;

void main() {
  vec2 texelSize = 1.0 / u_resolution;

  // 3x3 blur kernel
  float sum = 0.0;
  for (int x = -1; x <= 1; x++) {
    for (int y = -1; y <= 1; y++) {
      vec2 offset = vec2(float(x), float(y)) * texelSize;
      sum += texture2D(u_texture, v_texCoord + offset).r;
    }
  }

  float blurred = sum / 9.0;
  float decayed = blurred * u_decay;

  gl_FragColor = vec4(decayed, decayed, decayed, 1.0);
}
`

// Fragment shader for display with colors
const displayFragmentShader = `
precision mediump float;
uniform sampler2D u_texture;
uniform float u_time;
varying vec2 v_texCoord;

vec3 hsv2rgb(vec3 c) {
  vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
  vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
  return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

void main() {
  float intensity = texture2D(u_texture, v_texCoord).r;

  if (intensity < 0.01) {
    gl_FragColor = vec4(0.0, 0.0, 0.1, 1.0);
    return;
  }

  // Dynamic color based on intensity and position
  float hue = fract(intensity * 2.0 + v_texCoord.x * 0.1 + u_time * 0.05);
  float sat = 0.8 + intensity * 0.2;
  float val = min(intensity * 2.0, 1.0);

  vec3 color = hsv2rgb(vec3(hue, sat, val));

  gl_FragColor = vec4(color, 1.0);
}
`

function createShader(gl, type, source) {
  const shader = gl.createShader(type)
  gl.shaderSource(shader, source)
  gl.compileShader(shader)

  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error('Shader compilation error:', gl.getShaderInfoLog(shader))
    gl.deleteShader(shader)
    return null
  }

  return shader
}

function createProgram(gl, vertexShader, fragmentShader) {
  const program = gl.createProgram()
  gl.attachShader(program, vertexShader)
  gl.attachShader(program, fragmentShader)
  gl.linkProgram(program)

  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error('Program linking error:', gl.getProgramInfoLog(program))
    gl.deleteProgram(program)
    return null
  }

  return program
}

function createTexture(gl, width, height) {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, texture)

  const data = new Uint8Array(width * height * 4)
  data.fill(0)

  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, width, height, 0, gl.RGBA, gl.UNSIGNED_BYTE, data)

  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT)

  return texture
}

function createFramebuffer(gl, texture) {
  const framebuffer = gl.createFramebuffer()
  gl.bindFramebuffer(gl.FRAMEBUFFER, framebuffer)
  gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0)
  return framebuffer
}

class Particle {
  constructor(x, y) {
    this.x = x
    this.y = y
    this.heading = Math.random() * Math.PI * 2
  }

  sense(trailData) {
    const sd = props.sensorDistance
    const sa = props.sensorAngle

    const fx = this.x + Math.cos(this.heading) * sd
    const fy = this.y + Math.sin(this.heading) * sd
    const lx = this.x + Math.cos(this.heading + sa) * sd
    const ly = this.y + Math.sin(this.heading + sa) * sd
    const rx = this.x + Math.cos(this.heading - sa) * sd
    const ry = this.y + Math.sin(this.heading - sa) * sd

    const fw = this.sampleTrail(trailData, fx, fy)
    const lw = this.sampleTrail(trailData, lx, ly)
    const rw = this.sampleTrail(trailData, rx, ry)

    return { forward: fw, left: lw, right: rw }
  }

  sampleTrail(trailData, x, y) {
    const ix = Math.floor(x) % width
    const iy = Math.floor(y) % height
    const index = (iy * width + ix) * 4
    return trailData[index] / 255
  }

  update(trailData) {
    const { forward, left, right } = this.sense(trailData)

    // Steering logic
    if (forward > left && forward > right) {
      // Continue straight
    } else if (forward < left && forward < right) {
      // Random turn
      this.heading += (Math.random() - 0.5) * props.rotationAngle * 2
    } else if (left > right) {
      this.heading += props.rotationAngle
    } else {
      this.heading -= props.rotationAngle
    }

    // Mouse interaction
    if (isMouseDown) {
      const dx = mousePos.x - this.x
      const dy = mousePos.y - this.y
      const dist = Math.sqrt(dx * dx + dy * dy)
      if (dist < 150 && dist > 0) {
        const targetAngle = Math.atan2(dy, dx)
        const angleDiff = targetAngle - this.heading
        this.heading += Math.sin(angleDiff) * 0.1
      }
    }

    // Move
    this.x += Math.cos(this.heading) * props.moveDistance
    this.y += Math.sin(this.heading) * props.moveDistance

    // Wrap boundaries
    this.x = (this.x + width) % width
    this.y = (this.y + height) % height
  }
}

function initParticles() {
  particles = []
  const numParticles = Math.min(props.numParticles, 50000) // Cap for performance

  for (let i = 0; i < numParticles; i++) {
    // Initialize in center with some spread
    const angle = Math.random() * Math.PI * 2
    const radius = Math.random() * 100
    const x = width / 2 + Math.cos(angle) * radius
    const y = height / 2 + Math.sin(angle) * radius
    particles.push(new Particle(x, y))
  }
}

function setupWebGL() {
  gl = canvas.value.getContext('webgl') || canvas.value.getContext('experimental-webgl')

  if (!gl) {
    console.error('WebGL not supported')
    return false
  }

  // Create shaders
  const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource)
  const trailFS = createShader(gl, gl.FRAGMENT_SHADER, trailFragmentShader)
  const displayFS = createShader(gl, gl.FRAGMENT_SHADER, displayFragmentShader)

  if (!vertexShader || !trailFS || !displayFS) {
    return false
  }

  // Create programs
  trailProgram = createProgram(gl, vertexShader, trailFS)
  displayProgram = createProgram(gl, vertexShader, displayFS)

  if (!trailProgram || !displayProgram) {
    return false
  }

  // Create textures
  textures.trail = createTexture(gl, width, height)
  textures.trailTemp = createTexture(gl, width, height)

  // Create framebuffers
  frameBuffers.trail = createFramebuffer(gl, textures.trail)
  frameBuffers.trailTemp = createFramebuffer(gl, textures.trailTemp)

  // Create quad buffer
  const quadVertices = new Float32Array([
    -1, -1,  1, -1,  -1, 1,  1, 1
  ])

  buffers.quad = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.bufferData(gl.ARRAY_BUFFER, quadVertices, gl.STATIC_DRAW)

  // Enable blending for particle rendering
  gl.enable(gl.BLEND)
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)

  initParticles()

  return true
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  width = rect.width
  height = rect.height
  canvas.value.width = width
  canvas.value.height = height

  if (gl) {
    gl.viewport(0, 0, width, height)

    // Recreate textures with new size
    if (textures.trail) {
      gl.deleteTexture(textures.trail)
      gl.deleteTexture(textures.trailTemp)
      gl.deleteFramebuffer(frameBuffers.trail)
      gl.deleteFramebuffer(frameBuffers.trailTemp)
    }

    textures.trail = createTexture(gl, width, height)
    textures.trailTemp = createTexture(gl, width, height)
    frameBuffers.trail = createFramebuffer(gl, textures.trail)
    frameBuffers.trailTemp = createFramebuffer(gl, textures.trailTemp)

    initParticles()
  }
}

function render(timestamp) {
  if (!gl || !props.isPlaying) {
    if (props.isPlaying) {
      animationId = requestAnimationFrame(render)
    }
    return
  }

  time = timestamp * 0.001

  // Read current trail data for particle sensing
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail)
  const trailData = new Uint8Array(width * height * 4)
  gl.readPixels(0, 0, width, height, gl.RGBA, gl.UNSIGNED_BYTE, trailData)

  // Update particles
  particles.forEach(particle => particle.update(trailData))

  // Draw particles to trail texture
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trailTemp)
  gl.viewport(0, 0, width, height)

  // Copy current trail
  gl.useProgram(trailProgram)
  gl.uniform2f(gl.getUniformLocation(trailProgram, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(trailProgram, 'u_decay'), props.decayFactor)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(trailProgram, 'u_texture'), 0)

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  const posLoc = gl.getAttribLocation(trailProgram, 'a_position')
  gl.enableVertexAttribArray(posLoc)
  gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // Add particle deposits
  gl.enable(gl.BLEND)
  gl.blendFunc(gl.ONE, gl.ONE)

  // Draw particles as points
  const particleData = new Float32Array(particles.length * 2)
  for (let i = 0; i < particles.length; i++) {
    particleData[i * 2] = (particles[i].x / width) * 2 - 1
    particleData[i * 2 + 1] = (particles[i].y / height) * 2 - 1
  }

  const particleBuffer = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, particleBuffer)
  gl.bufferData(gl.ARRAY_BUFFER, particleData, gl.DYNAMIC_DRAW)

  // Simple point rendering (we'll use a minimal shader for this)
  gl.useProgram(displayProgram) // Reuse for simplicity
  gl.enableVertexAttribArray(posLoc)
  gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0)

  gl.uniform1f(gl.getUniformLocation(displayProgram, 'u_time'), time)
  gl.drawArrays(gl.POINTS, 0, particles.length)

  gl.deleteBuffer(particleBuffer)
  gl.disable(gl.BLEND)

  // Swap textures
  const tempTex = textures.trail
  const tempFB = frameBuffers.trail
  textures.trail = textures.trailTemp
  frameBuffers.trail = frameBuffers.trailTemp
  textures.trailTemp = tempTex
  frameBuffers.trailTemp = tempFB

  // Final display
  gl.bindFramebuffer(gl.FRAMEBUFFER, null)
  gl.viewport(0, 0, width, height)

  gl.useProgram(displayProgram)
  gl.uniform1f(gl.getUniformLocation(displayProgram, 'u_time'), time)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(displayProgram, 'u_texture'), 0)

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  const displayPosLoc = gl.getAttribLocation(displayProgram, 'a_position')
  gl.enableVertexAttribArray(displayPosLoc)
  gl.vertexAttribPointer(displayPosLoc, 2, gl.FLOAT, false, 0, 0)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  animationId = requestAnimationFrame(render)
}

function resetSimulation() {
  if (!gl) return

  // Clear trail textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trailTemp)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  // Reinitialize particles
  initParticles()
}

function onMouseDown(event) {
  isMouseDown = true
  updateMousePos(event)
}

function onMouseMove(event) {
  if (event.touches) return // Ignore mouse events during touch
  updateMousePos(event)
}

function onMouseUp() {
  isMouseDown = false
}

function onTouchStart(event) {
  event.preventDefault()
  isMouseDown = true
  updateMousePos(event.touches[0])
}

function onTouchMove(event) {
  event.preventDefault()
  updateMousePos(event.touches[0])
}

function onTouchEnd(event) {
  event.preventDefault()
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
  resizeCanvas()
  if (setupWebGL()) {
    emit('ready')
    animationId = requestAnimationFrame(render)
  }

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
  touch-action: none;
}
</style>
