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
  decayFactor: { type: Number, default: 0.95 },
  depositAmount: { type: Number, default: 5.0 },
  isPlaying: { type: Boolean, default: true }
})

let gl = null
let animationId = null
let programs = {}
let buffers = {}
let textures = {}
let frameBuffers = {}
let vaos = {}
let width = 0
let height = 0
let isMouseDown = false
let mousePos = { x: 0, y: 0 }
let time = 0
let particleTexSize = 0

const emit = defineEmits(['ready'])

// Vertex shader for full-screen quad
const quadVertexShader = `#version 300 es
precision highp float;

in vec2 a_position;
out vec2 v_texCoord;

void main() {
  gl_Position = vec4(a_position, 0.0, 1.0);
  v_texCoord = (a_position + 1.0) * 0.5;
}
`

// Particle update compute shader using transform feedback
const particleUpdateVertexShader = `#version 300 es
precision highp float;

in vec2 a_position;
in vec2 a_velocity;
in float a_angle;

uniform sampler2D u_trailTexture;
uniform vec2 u_resolution;
uniform float u_time;
uniform float u_deltaTime;
uniform float u_sensorDistance;
uniform float u_sensorAngle;
uniform float u_rotationAngle;
uniform float u_moveDistance;
uniform vec2 u_mousePos;
uniform float u_mouseInfluence;

out vec2 v_position;
out vec2 v_velocity;
out float v_angle;

// Random function
float hash(vec2 p) {
  return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
}

float sampleTrail(vec2 pos) {
  vec2 texCoord = pos / u_resolution;
  // Wrap coordinates
  texCoord = fract(texCoord + 1.0);
  return texture(u_trailTexture, texCoord).r;
}

void main() {
  vec2 pos = a_position;
  vec2 vel = a_velocity;
  float angle = a_angle;

  // Physarum sensing
  float sd = u_sensorDistance;
  float sa = u_sensorAngle;

  vec2 front = pos + vec2(cos(angle), sin(angle)) * sd;
  vec2 left = pos + vec2(cos(angle + sa), sin(angle + sa)) * sd;
  vec2 right = pos + vec2(cos(angle - sa), sin(angle - sa)) * sd;

  float frontWeight = sampleTrail(front);
  float leftWeight = sampleTrail(left);
  float rightWeight = sampleTrail(right);

  // Steering logic
  float randomFactor = hash(pos + u_time) - 0.5;

  if (frontWeight > leftWeight && frontWeight > rightWeight) {
    // Continue straight
  } else if (frontWeight < leftWeight && frontWeight < rightWeight) {
    // Random turn when forward is worst
    angle += randomFactor * u_rotationAngle * 2.0;
  } else if (leftWeight > rightWeight) {
    // Turn left
    angle += u_rotationAngle;
  } else {
    // Turn right
    angle -= u_rotationAngle;
  }

  // Mouse interaction
  vec2 toMouse = u_mousePos - pos;
  float mouseDist = length(toMouse);
  if (mouseDist > 0.0 && mouseDist < 150.0 && u_mouseInfluence > 0.0) {
    float mouseAngle = atan(toMouse.y, toMouse.x);
    float influence = u_mouseInfluence * (1.0 - mouseDist / 150.0);
    angle = mix(angle, mouseAngle, influence);
  }

  // Update velocity and position
  vec2 direction = vec2(cos(angle), sin(angle));
  vel = mix(vel, direction, 0.1); // Smooth velocity changes
  pos += vel * u_moveDistance;

  // Boundary wrapping
  pos = mod(pos + u_resolution, u_resolution);

  // Output new state
  v_position = pos;
  v_velocity = vel;
  v_angle = angle;

  // Convert to clip space for rendering
  gl_Position = vec4((pos / u_resolution) * 2.0 - 1.0, 0.0, 1.0);
  gl_PointSize = 1.0;
}
`

// Simple fragment shader for particles
const particleFragmentShader = `#version 300 es
precision mediump float;

out vec4 fragColor;

void main() {
  fragColor = vec4(1.0, 1.0, 1.0, 1.0);
}
`

// Trail diffusion and decay shader
const trailFragmentShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform sampler2D u_particleTexture;
uniform vec2 u_resolution;
uniform float u_decayFactor;
uniform float u_depositAmount;

in vec2 v_texCoord;
out vec4 fragColor;

void main() {
  vec2 texelSize = 1.0 / u_resolution;

  // 3x3 diffusion kernel
  float sum = 0.0;

  int index = 0;
  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 offset = vec2(float(x), float(y)) * texelSize;
      vec2 samplePos = fract(v_texCoord + offset + 1.0);

      float weight = 0.1;
      if (x == 0 && y == 0) weight = 0.2;
      else if (x == 0 || y == 0) weight = 0.125;
      else weight = 0.05;

      sum += texture(u_trailTexture, samplePos).r * weight;
    }
  }

  // Add new particle deposits
  float particleDeposit = texture(u_particleTexture, v_texCoord).r;

  // Apply decay and add deposit
  float newTrail = sum * u_decayFactor + particleDeposit * u_depositAmount;

  fragColor = vec4(newTrail, newTrail, newTrail, 1.0);
}
`

// Display shader with optimized color mapping
const displayFragmentShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform float u_time;
uniform vec2 u_resolution;

in vec2 v_texCoord;
out vec4 fragColor;

// Fast HSV to RGB conversion
vec3 hsv2rgb(vec3 c) {
  vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
  vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
  return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

void main() {
  float intensity = texture(u_trailTexture, v_texCoord).r;

  if (intensity < 0.001) {
    // Dark background with subtle gradient
    vec2 center = v_texCoord - 0.5;
    float vignette = 1.0 - length(center) * 0.8;
    fragColor = vec4(0.0, 0.0, 0.02 * vignette, 1.0);
    return;
  }

  // Dynamic color mapping
  float hue = fract(
    intensity * 1.2 +
    v_texCoord.x * 0.02 +
    v_texCoord.y * 0.01 +
    u_time * 0.01
  );

  float saturation = mix(0.6, 1.0, min(intensity * 2.0, 1.0));
  float brightness = min(intensity * 1.5, 1.0);

  // Enhanced brightness for high intensity areas
  if (intensity > 0.7) {
    brightness = mix(brightness, 1.2, (intensity - 0.7) * 3.33);
  }

  vec3 color = hsv2rgb(vec3(hue, saturation, brightness));

  fragColor = vec4(color, 1.0);
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

function createProgram(gl, vertexShader, fragmentShader, transformFeedbackVaryings = null) {
  const program = gl.createProgram()
  gl.attachShader(program, vertexShader)
  gl.attachShader(program, fragmentShader)

  if (transformFeedbackVaryings) {
    gl.transformFeedbackVaryings(program, transformFeedbackVaryings, gl.INTERLEAVED_ATTRIBS)
  }

  gl.linkProgram(program)

  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error('Program linking error:', gl.getProgramInfoLog(program))
    gl.deleteProgram(program)
    return null
  }

  return program
}

function createTexture(gl, width, height, internalFormat = gl.RGBA, format = gl.RGBA, type = gl.UNSIGNED_BYTE) {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, texture)

  gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, width, height, 0, format, type, null)

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

  if (gl.checkFramebufferStatus(gl.FRAMEBUFFER) !== gl.FRAMEBUFFER_COMPLETE) {
    console.error('Framebuffer not complete')
  }

  return framebuffer
}

function initParticles() {
  const numParticles = Math.min(props.numParticles, 500000)

  // Create particle data: position (2), velocity (2), angle (1) = 5 floats per particle
  const particleData = new Float32Array(numParticles * 5)

  for (let i = 0; i < numParticles; i++) {
    const idx = i * 5

    // Position - start in center with some spread
    const angle = Math.random() * Math.PI * 2
    const radius = Math.random() * 100
    particleData[idx] = width / 2 + Math.cos(angle) * radius
    particleData[idx + 1] = height / 2 + Math.sin(angle) * radius

    // Velocity - initial direction
    particleData[idx + 2] = Math.cos(angle)
    particleData[idx + 3] = Math.sin(angle)

    // Angle
    particleData[idx + 4] = angle
  }

  // Create transform feedback buffers (double buffered)
  buffers.particles1 = gl.createBuffer()
  buffers.particles2 = gl.createBuffer()

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.particles1)
  gl.bufferData(gl.ARRAY_BUFFER, particleData, gl.DYNAMIC_DRAW)

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.particles2)
  gl.bufferData(gl.ARRAY_BUFFER, particleData, gl.DYNAMIC_DRAW)

  return numParticles
}

function setupWebGL() {
  gl = canvas.value.getContext('webgl2', {
    antialias: false,
    depth: false,
    stencil: false,
    alpha: false,
    preserveDrawingBuffer: false,
    powerPreference: 'high-performance'
  })

  if (!gl) {
    console.error('WebGL2 not supported')
    return false
  }

  // Check for required extensions
  const floatTexExt = gl.getExtension('EXT_color_buffer_float')
  if (!floatTexExt) {
    console.warn('Float textures not supported, using lower precision')
  }

  // Create shaders
  const quadVS = createShader(gl, gl.VERTEX_SHADER, quadVertexShader)
  const particleUpdateVS = createShader(gl, gl.VERTEX_SHADER, particleUpdateVertexShader)
  const particleFS = createShader(gl, gl.FRAGMENT_SHADER, particleFragmentShader)
  const trailFS = createShader(gl, gl.FRAGMENT_SHADER, trailFragmentShader)
  const displayFS = createShader(gl, gl.FRAGMENT_SHADER, displayFragmentShader)

  if (!quadVS || !particleUpdateVS || !particleFS || !trailFS || !displayFS) {
    return false
  }

  // Create programs
  programs.particleUpdate = createProgram(gl, particleUpdateVS, particleFS,
    ['v_position', 'v_velocity', 'v_angle'])
  programs.trail = createProgram(gl, quadVS, trailFS)
  programs.display = createProgram(gl, quadVS, displayFS)

  if (!programs.particleUpdate || !programs.trail || !programs.display) {
    return false
  }

  // Initialize particles
  const numParticles = initParticles()

  // Create textures
  const useFloatTextures = floatTexExt !== null
  const internalFormat = useFloatTextures ? gl.R32F : gl.RGBA
  const format = useFloatTextures ? gl.RED : gl.RGBA
  const type = useFloatTextures ? gl.FLOAT : gl.UNSIGNED_BYTE

  textures.trail1 = createTexture(gl, width, height, internalFormat, format, type)
  textures.trail2 = createTexture(gl, width, height, internalFormat, format, type)
  textures.particles = createTexture(gl, width, height, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE)

  // Create framebuffers
  frameBuffers.trail1 = createFramebuffer(gl, textures.trail1)
  frameBuffers.trail2 = createFramebuffer(gl, textures.trail2)
  frameBuffers.particles = createFramebuffer(gl, textures.particles)

  // Create quad buffer
  const quadVertices = new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1])
  buffers.quad = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.bufferData(gl.ARRAY_BUFFER, quadVertices, gl.STATIC_DRAW)

  // Create VAOs for better performance
  vaos.quad = gl.createVertexArray()
  gl.bindVertexArray(vaos.quad)
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0)

  vaos.particles1 = gl.createVertexArray()
  gl.bindVertexArray(vaos.particles1)
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.particles1)
  gl.enableVertexAttribArray(0) // position
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 20, 0)
  gl.enableVertexAttribArray(1) // velocity
  gl.vertexAttribPointer(1, 2, gl.FLOAT, false, 20, 8)
  gl.enableVertexAttribArray(2) // angle
  gl.vertexAttribPointer(2, 1, gl.FLOAT, false, 20, 16)

  vaos.particles2 = gl.createVertexArray()
  gl.bindVertexArray(vaos.particles2)
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.particles2)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 20, 0)
  gl.enableVertexAttribArray(1)
  gl.vertexAttribPointer(1, 2, gl.FLOAT, false, 20, 8)
  gl.enableVertexAttribArray(2)
  gl.vertexAttribPointer(2, 1, gl.FLOAT, false, 20, 16)

  // Clear initial textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail1)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindVertexArray(null)

  return { numParticles }
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  const pixelRatio = Math.min(window.devicePixelRatio || 1, 1.5) // Limit pixel ratio for performance

  width = Math.floor(rect.width * pixelRatio)
  height = Math.floor(rect.height * pixelRatio)

  canvas.value.width = width
  canvas.value.height = height
  canvas.value.style.width = rect.width + 'px'
  canvas.value.style.height = rect.height + 'px'

  if (gl) {
    gl.viewport(0, 0, width, height)

    // Recreate textures and framebuffers with new size
    if (textures.trail1) {
      gl.deleteTexture(textures.trail1)
      gl.deleteTexture(textures.trail2)
      gl.deleteTexture(textures.particles)
      gl.deleteFramebuffer(frameBuffers.trail1)
      gl.deleteFramebuffer(frameBuffers.trail2)
      gl.deleteFramebuffer(frameBuffers.particles)

      const floatTexExt = gl.getExtension('EXT_color_buffer_float')
      const useFloatTextures = floatTexExt !== null
      const internalFormat = useFloatTextures ? gl.R32F : gl.RGBA
      const format = useFloatTextures ? gl.RED : gl.RGBA
      const type = useFloatTextures ? gl.FLOAT : gl.UNSIGNED_BYTE

      textures.trail1 = createTexture(gl, width, height, internalFormat, format, type)
      textures.trail2 = createTexture(gl, width, height, internalFormat, format, type)
      textures.particles = createTexture(gl, width, height, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE)

      frameBuffers.trail1 = createFramebuffer(gl, textures.trail1)
      frameBuffers.trail2 = createFramebuffer(gl, textures.trail2)
      frameBuffers.particles = createFramebuffer(gl, textures.particles)
    }
  }
}

let lastTime = 0
let currentParticleBuffer = 0
let numParticles = 0

function render(timestamp) {
  if (!gl || !props.isPlaying) {
    if (props.isPlaying) {
      animationId = requestAnimationFrame(render)
    }
    return
  }

  const deltaTime = Math.min((timestamp - lastTime) * 0.001, 0.016) // Cap delta time
  lastTime = timestamp
  time = timestamp * 0.001

  // 1. Update particles using transform feedback
  gl.useProgram(programs.particleUpdate)

  // Set uniforms
  gl.uniform2f(gl.getUniformLocation(programs.particleUpdate, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_time'), time)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_deltaTime'), deltaTime)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_sensorDistance'), props.sensorDistance)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_sensorAngle'), props.sensorAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_rotationAngle'), props.rotationAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_moveDistance'), props.moveDistance)
  gl.uniform2f(gl.getUniformLocation(programs.particleUpdate, 'u_mousePos'), mousePos.x, mousePos.y)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_mouseInfluence'), isMouseDown ? 0.1 : 0.0)

  // Bind trail texture for sensing
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.particleUpdate, 'u_trailTexture'), 0)

  // Setup transform feedback
  const inputVAO = currentParticleBuffer === 0 ? vaos.particles1 : vaos.particles2
  const outputBuffer = currentParticleBuffer === 0 ? buffers.particles2 : buffers.particles1

  // Unbind any existing buffer bindings to avoid conflicts
  gl.bindBuffer(gl.ARRAY_BUFFER, null)
  gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, null)

  // Bind input VAO for reading particle data
  gl.bindVertexArray(inputVAO)

  // Bind output buffer for transform feedback writing
  gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, outputBuffer)

  // Render particles to trail texture
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particles)
  gl.viewport(0, 0, width, height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.enable(gl.BLEND)
  gl.blendFunc(gl.ONE, gl.ONE)

  // Disable rasterizer since we only want transform feedback
  gl.enable(gl.RASTERIZER_DISCARD)

  gl.beginTransformFeedback(gl.POINTS)
  gl.drawArrays(gl.POINTS, 0, numParticles)
  gl.endTransformFeedback()

  // Re-enable rasterizer
  gl.disable(gl.RASTERIZER_DISCARD)

  // Unbind transform feedback buffer to avoid conflicts
  gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, null)

  // Now render particles as points for visualization
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particles)
  gl.viewport(0, 0, width, height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  // Use the updated particle data (output buffer) for rendering
  const renderVAO = currentParticleBuffer === 0 ? vaos.particles2 : vaos.particles1
  gl.bindVertexArray(renderVAO)

  gl.enable(gl.BLEND)
  gl.blendFunc(gl.ONE, gl.ONE)

  gl.drawArrays(gl.POINTS, 0, numParticles)

  gl.disable(gl.BLEND)
  gl.bindVertexArray(null)

  // Swap particle buffers
  currentParticleBuffer = 1 - currentParticleBuffer

  // 2. Trail processing (diffusion and decay)
  gl.useProgram(programs.trail)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.viewport(0, 0, width, height)

  gl.uniform2f(gl.getUniformLocation(programs.trail, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_decayFactor'), props.decayFactor)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_depositAmount'), props.depositAmount)

  // Bind textures
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_trailTexture'), 0)

  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.particles)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_particleTexture'), 1)

  // Render full-screen quad
  gl.bindVertexArray(vaos.quad)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // Swap trail textures
  const tempTex = textures.trail1
  const tempFB = frameBuffers.trail1
  textures.trail1 = textures.trail2
  frameBuffers.trail1 = frameBuffers.trail2
  textures.trail2 = tempTex
  frameBuffers.trail2 = tempFB

  // 3. Final display
  gl.useProgram(programs.display)
  gl.bindFramebuffer(gl.FRAMEBUFFER, null)
  gl.viewport(0, 0, width, height)

  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_time'), time)
  gl.uniform2f(gl.getUniformLocation(programs.display, 'u_resolution'), width, height)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_trailTexture'), 0)

  gl.bindVertexArray(vaos.quad)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  animationId = requestAnimationFrame(render)
}

function resetSimulation() {
  if (!gl) return

  time = 0
  lastTime = 0

  // Clear trail textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail1)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  // Reinitialize particles
  numParticles = initParticles()
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
  const pixelRatio = Math.min(window.devicePixelRatio || 1, 1.5)

  mousePos.x = (event.clientX - rect.left) * pixelRatio
  mousePos.y = height - (event.clientY - rect.top) * pixelRatio // Flip Y coordinate
}

watch(() => props.isPlaying, (isPlaying) => {
  if (isPlaying && !animationId) {
    animationId = requestAnimationFrame(render)
  }
})

onMounted(() => {
  resizeCanvas()
  const result = setupWebGL()
  if (result) {
    numParticles = result.numParticles
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
  background: #000;
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
