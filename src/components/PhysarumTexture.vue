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
  numParticles: { type: Number, default: 50000 },
  sensorDistance: { type: Number, default: 9.0 },
  sensorAngle: { type: Number, default: 0.5 },
  rotationAngle: { type: Number, default: 0.3 },
  moveDistance: { type: Number, default: 1.0 },
  decayFactor: { type: Number, default: 0.95 },
  depositAmount: { type: Number, default: 5.0 },
  isPlaying: { type: Boolean, default: true },
  resolution: { type: Number, default: 0.3 }
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
let numParticles = 0

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

// Particle simulation using texture-based approach
const particleUpdateShader = `#version 300 es
precision highp float;

uniform sampler2D u_particleData;
uniform sampler2D u_trailTexture;
uniform vec2 u_resolution;
uniform vec2 u_particleTexSize;
uniform float u_time;
uniform float u_sensorDistance;
uniform float u_sensorAngle;
uniform float u_rotationAngle;
uniform float u_moveDistance;
uniform vec2 u_mousePos;
uniform float u_mouseInfluence;

in vec2 v_texCoord;
out vec4 fragColor;

// Hash function for randomness
float hash(vec2 p) {
  return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
}

float sampleTrail(vec2 pos) {
  vec2 texCoord = pos / u_resolution;
  texCoord = fract(texCoord + 1.0);
  return texture(u_trailTexture, texCoord).r;
}

void main() {
  // Get current particle data
  vec4 particleData = texture(u_particleData, v_texCoord);
  
  vec2 pos = particleData.xy;
  float angle = particleData.z;
  float unused = particleData.w; // For future use
  
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
  if (u_mouseInfluence > 0.0) {
    vec2 toMouse = u_mousePos - pos;
    float mouseDist = length(toMouse);
    if (mouseDist > 0.0 && mouseDist < 150.0) {
      float mouseAngle = atan(toMouse.y, toMouse.x);
      float influence = u_mouseInfluence * (1.0 - mouseDist / 150.0);
      angle = mix(angle, mouseAngle, influence);
    }
  }
  
  // Update position
  pos += vec2(cos(angle), sin(angle)) * u_moveDistance;
  
  // Boundary wrapping
  pos = mod(pos + u_resolution, u_resolution);
  
  // Output new particle state
  fragColor = vec4(pos, angle, 0.0);
}
`

// Optimized particle vertex shader for point sprites
const particleVertexShader = `#version 300 es
precision highp float;

uniform sampler2D u_particleData;
uniform vec2 u_resolution;
uniform vec2 u_particleTexSize;

out float v_intensity;

void main() {
  // Calculate texture coordinate from vertex ID
  int particleId = gl_VertexID;
  int texWidth = int(u_particleTexSize.x);
  int x = particleId % texWidth;
  int y = particleId / texWidth;
  
  vec2 texCoord = (vec2(float(x), float(y)) + 0.5) / u_particleTexSize;
  
  // Sample particle data
  vec4 particleData = texture(u_particleData, texCoord);
  vec2 position = particleData.xy;
  
  // Convert to clip space
  vec2 clipPos = (position / u_resolution) * 2.0 - 1.0;
  clipPos.y = -clipPos.y; // Flip Y for screen space
  
  gl_Position = vec4(clipPos, 0.0, 1.0);
  gl_PointSize = 1.0;
  
  v_intensity = 1.0;
}
`

// Simplified particle fragment shader
const particleFragmentShader = `#version 300 es
precision highp float;

in float v_intensity;
out vec4 fragColor;

void main() {
  fragColor = vec4(v_intensity, 0.0, 0.0, 1.0);
}
`

// Trail diffusion and decay shader  
const trailShader = `#version 300 es
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
  
  // 3x3 diffusion
  float sum = 0.0;
  
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
  
  // Add particle deposits
  float particleDeposit = texture(u_particleTexture, v_texCoord).r;
  
  // Apply decay and add deposit
  float newTrail = sum * u_decayFactor + particleDeposit * u_depositAmount;
  
  fragColor = vec4(newTrail, newTrail, newTrail, 1.0);
}
`

// Display shader
const displayShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform float u_time;

in vec2 v_texCoord;
out vec4 fragColor;

vec3 hsv2rgb(vec3 c) {
  vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
  vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
  return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

void main() {
  float intensity = texture(u_trailTexture, v_texCoord).r;
  
  if (intensity < 0.001) {
    vec2 center = v_texCoord - 0.5;
    float vignette = 1.0 - length(center) * 0.8;
    fragColor = vec4(0.0, 0.0, 0.02 * vignette, 1.0);
    return;
  }
  
  float hue = fract(
    intensity * 1.2 + 
    v_texCoord.x * 0.02 + 
    v_texCoord.y * 0.01 + 
    u_time * 0.01
  );
  
  float saturation = mix(0.6, 1.0, min(intensity * 2.0, 1.0));
  float brightness = min(intensity * 1.5, 1.0);
  
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

function createTexture(gl, width, height, internalFormat = gl.RGBA, format = gl.RGBA, type = gl.UNSIGNED_BYTE, data = null) {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, texture)
  
  gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, width, height, 0, format, type, data)
  
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST)
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
  numParticles = Math.min(props.numParticles, 50000) // Reduced limit for better performance
  
  // Calculate texture size to hold particle data
  const particleTexSize = Math.ceil(Math.sqrt(numParticles))
  
  // Create initial particle data
  const particleData = new Float32Array(particleTexSize * particleTexSize * 4)
  
  for (let i = 0; i < numParticles; i++) {
    const idx = i * 4
    
    // Position - start in center with some spread
    const angle = Math.random() * Math.PI * 2
    const radius = Math.random() * 100
    particleData[idx] = width / 2 + Math.cos(angle) * radius       // x
    particleData[idx + 1] = height / 2 + Math.sin(angle) * radius  // y
    particleData[idx + 2] = angle                                   // angle
    particleData[idx + 3] = 0                                      // unused
  }
  
  // Create particle data textures (double buffered)
  textures.particles1 = createTexture(gl, particleTexSize, particleTexSize, 
    gl.RGBA32F, gl.RGBA, gl.FLOAT, particleData)
  textures.particles2 = createTexture(gl, particleTexSize, particleTexSize, 
    gl.RGBA32F, gl.RGBA, gl.FLOAT, particleData)
  
  frameBuffers.particles1 = createFramebuffer(gl, textures.particles1)
  frameBuffers.particles2 = createFramebuffer(gl, textures.particles2)
  
  return particleTexSize
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
  
  const floatTexExt = gl.getExtension('EXT_color_buffer_float')
  if (!floatTexExt) {
    console.error('Float textures not supported')
    return false
  }
  
  // Create shaders
  const quadVS = createShader(gl, gl.VERTEX_SHADER, quadVertexShader)
  const particleVS = createShader(gl, gl.VERTEX_SHADER, particleVertexShader)
  const particleUpdateFS = createShader(gl, gl.FRAGMENT_SHADER, particleUpdateShader)
  const particleRenderFS = createShader(gl, gl.FRAGMENT_SHADER, particleFragmentShader)
  const trailFS = createShader(gl, gl.FRAGMENT_SHADER, trailShader)
  const displayFS = createShader(gl, gl.FRAGMENT_SHADER, displayShader)
  
  if (!quadVS || !particleVS || !particleUpdateFS || !particleRenderFS || !trailFS || !displayFS) {
    return false
  }
  
  // Create programs
  programs.particleUpdate = createProgram(gl, quadVS, particleUpdateFS)
  programs.particleRender = createProgram(gl, particleVS, particleRenderFS)
  programs.trail = createProgram(gl, quadVS, trailFS)
  programs.display = createProgram(gl, quadVS, displayFS)
  
  if (!programs.particleUpdate || !programs.particleRender || !programs.trail || !programs.display) {
    return false
  }
  
  // Initialize particles
  const particleTexSize = initParticles()
  
  // Create trail textures
  textures.trail1 = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
  textures.trail2 = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
  textures.particleRender = createTexture(gl, width, height, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE)
  
  frameBuffers.trail1 = createFramebuffer(gl, textures.trail1)
  frameBuffers.trail2 = createFramebuffer(gl, textures.trail2)
  frameBuffers.particleRender = createFramebuffer(gl, textures.particleRender)
  
  // Create quad buffer and VAO
  const quadVertices = new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1])
  buffers.quad = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.bufferData(gl.ARRAY_BUFFER, quadVertices, gl.STATIC_DRAW)
  
  vaos.quad = gl.createVertexArray()
  gl.bindVertexArray(vaos.quad)
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0)
  
  // Create dummy vertex array for particle rendering (using gl_VertexID)
  vaos.particles = gl.createVertexArray()
  gl.bindVertexArray(vaos.particles)
  // No attributes needed since we use gl_VertexID
  
  gl.bindVertexArray(null)
  
  // Clear initial textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail1)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)
  
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)
  
  return { particleTexSize }
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  const pixelRatio = Math.min(window.devicePixelRatio || 1, 1)
  
  // Use resolution parameter to scale down rendering resolution
  width = Math.floor(rect.width * pixelRatio * props.resolution)
  height = Math.floor(rect.height * pixelRatio * props.resolution)
  
  // Ensure minimum resolution
  width = Math.max(width, 256)
  height = Math.max(height, 256)
  
  canvas.value.width = width
  canvas.value.height = height
  canvas.value.style.width = rect.width + 'px'
  canvas.value.style.height = rect.height + 'px'
  
  if (gl) {
    gl.viewport(0, 0, width, height)
    
    // Recreate screen-sized textures
    if (textures.trail1) {
      gl.deleteTexture(textures.trail1)
      gl.deleteTexture(textures.trail2)
      gl.deleteTexture(textures.particleRender)
      gl.deleteFramebuffer(frameBuffers.trail1)
      gl.deleteFramebuffer(frameBuffers.trail2)
      gl.deleteFramebuffer(frameBuffers.particleRender)
      
      textures.trail1 = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
      textures.trail2 = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
      textures.particleRender = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
      
      frameBuffers.trail1 = createFramebuffer(gl, textures.trail1)
      frameBuffers.trail2 = createFramebuffer(gl, textures.trail2)
      frameBuffers.particleRender = createFramebuffer(gl, textures.particleRender)
    }
  }
}

let lastTime = 0
let currentParticleBuffer = 0
let particleTexSize = 0

function render(timestamp) {
  if (!gl || !props.isPlaying) {
    if (props.isPlaying) {
      animationId = requestAnimationFrame(render)
    }
    return
  }
  
  const deltaTime = Math.min((timestamp - lastTime) * 0.001, 0.016)
  lastTime = timestamp
  time = timestamp * 0.001
  
  // 1. Update particles
  gl.useProgram(programs.particleUpdate)
  
  const inputParticleTexture = currentParticleBuffer === 0 ? textures.particles1 : textures.particles2
  const outputFramebuffer = currentParticleBuffer === 0 ? frameBuffers.particles2 : frameBuffers.particles1
  
  gl.bindFramebuffer(gl.FRAMEBUFFER, outputFramebuffer)
  gl.viewport(0, 0, particleTexSize, particleTexSize)
  
  gl.uniform2f(gl.getUniformLocation(programs.particleUpdate, 'u_resolution'), width, height)
  gl.uniform2f(gl.getUniformLocation(programs.particleUpdate, 'u_particleTexSize'), particleTexSize, particleTexSize)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_time'), time)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_sensorDistance'), props.sensorDistance)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_sensorAngle'), props.sensorAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_rotationAngle'), props.rotationAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_moveDistance'), props.moveDistance)
  gl.uniform2f(gl.getUniformLocation(programs.particleUpdate, 'u_mousePos'), mousePos.x, mousePos.y)
  gl.uniform1f(gl.getUniformLocation(programs.particleUpdate, 'u_mouseInfluence'), isMouseDown ? 0.1 : 0.0)
  
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, inputParticleTexture)
  gl.uniform1i(gl.getUniformLocation(programs.particleUpdate, 'u_particleData'), 0)
  
  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.particleUpdate, 'u_trailTexture'), 1)
  
  gl.bindVertexArray(vaos.quad)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)
  
  currentParticleBuffer = 1 - currentParticleBuffer
  
  // 2. Render particles using point sprites
  gl.useProgram(programs.particleRender)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particleRender)
  gl.viewport(0, 0, width, height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)
  
  gl.uniform2f(gl.getUniformLocation(programs.particleRender, 'u_resolution'), width, height)
  gl.uniform2f(gl.getUniformLocation(programs.particleRender, 'u_particleTexSize'), particleTexSize, particleTexSize)
  
  const currentParticleTexture = currentParticleBuffer === 0 ? textures.particles1 : textures.particles2
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, currentParticleTexture)
  gl.uniform1i(gl.getUniformLocation(programs.particleRender, 'u_particleData'), 0)
  
  // Enable point sprites and render particles
  gl.bindVertexArray(vaos.particles)
  gl.enable(gl.BLEND)
  gl.blendFunc(gl.ONE, gl.ONE)
  gl.drawArrays(gl.POINTS, 0, numParticles)
  gl.disable(gl.BLEND)
  
  // 3. Update trails
  gl.useProgram(programs.trail)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.viewport(0, 0, width, height)
  
  gl.uniform2f(gl.getUniformLocation(programs.trail, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_decayFactor'), props.decayFactor)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_depositAmount'), props.depositAmount)
  
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_trailTexture'), 0)
  
  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.particleRender)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_particleTexture'), 1)
  
  gl.bindVertexArray(vaos.quad)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)
  
  // Swap trail textures
  const tempTex = textures.trail1
  const tempFB = frameBuffers.trail1
  textures.trail1 = textures.trail2
  frameBuffers.trail1 = frameBuffers.trail2
  textures.trail2 = tempTex
  frameBuffers.trail2 = tempFB
  
  // 4. Final display
  gl.useProgram(programs.display)
  gl.bindFramebuffer(gl.FRAMEBUFFER, null)
  gl.viewport(0, 0, width, height)
  
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_time'), time)
  
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
  
  // Delete old particle textures if they exist
  if (textures.particles1) {
    gl.deleteTexture(textures.particles1)
    gl.deleteTexture(textures.particles2)
    gl.deleteFramebuffer(frameBuffers.particles1)
    gl.deleteFramebuffer(frameBuffers.particles2)
  }
  
  // Reinitialize particles with current settings
  particleTexSize = initParticles()
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
  mousePos.y = height - (event.clientY - rect.top) * pixelRatio
}

watch(() => props.isPlaying, (isPlaying) => {
  if (isPlaying && !animationId) {
    animationId = requestAnimationFrame(render)
  }
})

// Watch for resolution changes and resize canvas
watch(() => props.resolution, () => {
  if (gl) {
    resizeCanvas()
  }
})

// Watch for numParticles changes and reinitialize
watch(() => props.numParticles, () => {
  if (gl) {
    resetSimulation()
  }
})

onMounted(() => {
  resizeCanvas()
  const result = setupWebGL()
  if (result) {
    particleTexSize = result.particleTexSize
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
